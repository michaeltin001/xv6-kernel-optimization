# Copy-on-Write Fork

## Background

> The `fork()` system call in Xv6 copies all of the parent process's user-space memory into the child.
If the parent is large, copying can take a long time. In addition, the copies often waste memory; in many cases neither the parent nor the child modifies a page, so that in principle they could share the same physical memory.
The inefficiency is particularly clear if the child calls `exec()`, since `exec()` will throw away the copied pages, probably without using most of them. On the other hand, if both parent and child use a page, and one or both writes it, a copy is truly needed.
>
> The goal of copy-on-write (COW) `fork()` is to defer allocating and copying physical memory pages for the child until the copies are actually needed, if ever.
>
>> COW `fork()` creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages.
>>
>> COW `fork()` marks all the user PTEs in both parent and child as not writable. When either process tries to write one of these COW pages, the CPU will force a page fault.
>>
>>  The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable.
>>
>> When the page fault handler returns, the user process will be able to write its copy of the page.
>
> COW `fork()` makes freeing of the physical pages that implement user memory a little trickier. A given physical page may be referred to by multiple processes' page tables, and should be freed only when the last reference disappears.

## Implement Page Reference Counter

### Objective

> In `kalloc.c`, a new structure should be created to record the reference count of each physical page. This can be implemented using a data structure such as a linked list or a fixed-length array. Any modifications to the reference count must be protected by a lock to ensure thread-safe updates. When the `kalloc()` function allocates a new physical page, the reference count for that page should be initialized to 1. Within the `kfree()` function, the reference count should be decremented, and the physical page should only be released when the count reaches 0. Additionally, a separate function should be implemented to increment the reference count when a page gains an additional reference.

### Implementation

The implementation of the page reference counter is straightforward. As mentioned in the objective, we need to modify the functions in `kalloc.c`. We first modify the `kmem` struct to include a reference counter. This is a fixed length array that allows us to access the reference count of each physical page. The size of the array is defined as `PHYMEM / PGSIZE`, where:

* `PHYMEM = PHYSTOP - KERNBASE`. This is the last physical memory address minus the beginning of the kernel memory, which gives us the total physical memory. This is also equivalent to `128 * 1024 * 1024`.

  * `PHYMEM` is a custom definition.
* `PGSIZE` is the page size, which is `4096`.

#### `(kernel/kalloc.c)`

```c
#define PHYMEM (PHYSTOP - KERNBASE) // PHYSTOP - KERNBASE = 128 * 1024 * 1024
...
struct {
  struct spinlock lock;
  struct run *freelist;
  int refcnt[PHYMEM / PGSIZE]; // Fixed length array to access the reference count.
} kmem;
```

We add some functionality to the `kfree()` function.
The reference count is decremented; then, we check if it has reached 0.

* If the reference count is greater than 0, we return and do not execute the rest of the code.
* If the reference count is 0, we release the physical page.
  Modification of the reference count is guarded by the lock.

#### `kfree()` (`kernel/kalloc.c`)

```c
void
kfree(void *pa)
{
  ...
  // Decrement the ref count, release the physical page when it reaches 0.
  acquire(&kmem.lock); // Acquire lock.
  uint64 index = (((uint64)pa - KERNBASE) / PGSIZE); // Calculate the page index.
  kmem.refcnt[index]--; // Decrement the ref count.
  if (kmem.refcnt[index] > 0) { // If physical page has not reached 0, return early.
    release(&kmem.lock); // Release lock.
    return;
  }
  release(&kmem.lock); // Release lock.
  ...
}
```

Additionally, we add functionality to `kalloc()` to set all newly allocated physical page reference count to 1.

#### `kalloc()` (`kernel/kalloc.c`)

```c
void *
kalloc(void)
{
  struct run *r;
  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
    uint64 index = (((uint64)r - KERNBASE) / PGSIZE); // Calculate the page index.
    kmem.refcnt[index] = 1; // Newly allocated physical page ref count = 1.
  }
  release(&kmem.lock);
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

We create two new functions to handle incrementing and decrementing of the reference count. Since modification of the reference count is guarded by the lock, adding these two new functions makes things easier when executing this code in other files.

The incrementing function, `refinc()`, is straightforward. It simply passes in a physical page address and then increments its reference count.

#### `refinc()` (`kernel/kalloc.c`)

```c
// New function to increment the ref count.
void
refinc(uint64 pa)
{
  acquire(&kmem.lock); // Acquire lock.
  uint64 index = (((uint64)pa - KERNBASE) / PGSIZE); // Calculate the page index.
  kmem.refcnt[index]++; // Increment the ref count.
  release(&kmem.lock); // Release lock.
}
```

The decrementing function, `refdec()`, is also straightforward. It passes in a physical page address and decrements its reference count. However, we add one extra check to this function, which is to call `kfree()` on the physical page if the reference count reaches 0.

#### `refdec()` (`kernel/kalloc.c`)

```c
// New function to decrement the ref count.
void
refdec(uint64 pa)
{
  acquire(&kmem.lock); // Acquire lock.
  uint64 index = (((uint64)pa - KERNBASE) / PGSIZE); // Calculate the page index.
  kmem.refcnt[index]--; // Decrement the ref count.
  if (kmem.refcnt[index] == 0) {
    release(&kmem.lock); // Release lock.
    kfree((void*)pa);
  }
  else {
    release(&kmem.lock); // Release lock.
  }
}
```

We add these new functions to the declarations for `kalloc.c` in `defs.h`.

#### `(kernel/defs.h)`

```c
// kalloc.c
void* kalloc(void);
void kfree(void *);
void kinit();
void refinc(uint64);
void refdec(uint64);
```

## Fix `uvmcopy()` function

### Objective

> The `uvmcopy()` function should be modified to support copy-on-write behavior. Instead of allocating a new physical page for the child process, the child’s virtual page should be mapped to the same physical page used by the parent. The write permission (`PTE_W`) should be cleared from the page table entries of both processes to prevent direct modification of the shared page. A new privilege flag should also be defined to indicate that a page table entry represents a copy-on-write (COW) mapping; this flag can be defined in `riscv.h`. The reference counter for the shared physical page should be incremented to reflect that the page is now referenced by multiple processes.

### Implementation

We first start by defining a privilege flag to record whether a PTE is COW mapping. This flag will be known as `PTE_COW`.

In Chapter 3.1 of the XV6 book, we can observe the RISC-V page table hardware. We see that Bits 0 - 7 are reserved for other functions, so we choose Bit 8 for the new `PTE_COW` privilege flag. We then add the privilege flag to `riscv.h`.

#### `(kernel/riscv.h)`

```c
#define PTE_COW (1L << 8)
```

The modifications to `uvmcopy()` itself are as follows:

1. Clear `PTE_W` for the parent and child PTE.
2. Record `PTE_COW` for the parent and child PTE.
3. Map the parent’s physical page to the child’s virtual page.
4. Increment the page reference counter of this physical page.

We also remove the functionality of new page allocation (commented out).

#### `uvmcopy()` (`kernel/vm.c`)

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  // char *mem;
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    // 1. Clear PTE_W for the parent and child PTE.
    *pte &= (~PTE_W);
    // 2. Record PTE_COW for the parent and child PTE.
    *pte |= PTE_COW;
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    // Remove the new page allocation.
    // if((mem = kalloc()) == 0)
    //   goto err;
    // memmove(mem, (char*)pa, PGSIZE);
    // if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
    //   kfree(mem);
    //   goto err;
    // }
    // 3. Map parent's physical page to child's virtual page.
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0)
      goto err;
    // 4. Increment the ref count of this physical page.
    refinc(pa);
  }
  return 0;
err:
  uvmunmap(new, 0, i, 1);
  return -1;
}
```

## Fix `usertrap()` function

### Objective

> The `usertrap()` function should be modified to properly handle write page faults that occur on copy-on-write (COW) pages. When a page fault occurs, the handler should check whether the cause corresponds to a write fault (i.e., `r_scause()` returns 15) and verify that the faulting page is marked as a COW page using the newly defined privilege flag. If this condition is met, the kernel should allocate a new physical page and duplicate the contents of the original COW page into the new page, which can be performed using `memmove()`. After the copy is completed, the faulting virtual page should be remapped to the newly allocated physical page with the write permission (`PTE_W`) enabled. This remapping involves unmapping the original COW page and then mapping the virtual address to the new physical page with the updated permissions.

### Implementation

The `usertrap()` function needs to 1) detect a page fault, 2) allocate physical memory, 3) copy the original page into the new page, and then 4) modify the relevant PTE to refer to the new page.

The implementation is as follows:

1. Add an “else if” branch to check for page faults.
   a. `else if (r_scause() == 15)`
2. See if the virtual address is valid (if not, kill the process).
3. Traverse the page table and get a pointer to the PTE that corresponds to the virtual address. If the PTE is null, kill the process.
4. Check if the `PTE_COW` flag is set in the PTE.
5. Extract the privilege flags from the PTE.
6. Record the `PTE_W` privilege flag.
7. Clear the `PTE_COW` privilege flag.
8. Allocate a new page of physical memory using `kalloc()`. If `kalloc()` returns null, kill the process (memory allocation failed).
9. Extract the physical address from the original PTE.
10. Using `memmove()`, copy the contents of the original page to the new page.
11. Decrease the reference count of the original page.
12. Unmap from the original page.
13. Map to the new page.

#### `usertrap()` (`kernel/trap.c`)

```c
void
usertrap(void)
{
  ...
  else if (r_scause() == 15) {
    uint64 va = PGROUNDDOWN(r_stval());
    if (va >= MAXVA) {
      p->killed = 1;
      goto end;
    }
    pte_t *pte = walk(p->pagetable, va, 0);
    if (!pte) {
      p->killed = 1;
      goto end;
    }
    if (*pte & PTE_COW) {
      uint flags = PTE_FLAGS(*pte);
      flags |= PTE_W;
      flags &= (~PTE_COW);
      char *mem = kalloc();
      if (!mem) {
        p->killed = 1;
        goto end;
      }
      uint64 pa = (uint64)PTE2PA(*pte);
      memmove(mem, (void*)pa, PGSIZE);
      refdec(pa);
      uvmunmap(p->pagetable, va, PGSIZE, 0);
      if (mappages(p->pagetable, va, PGSIZE, (uint64)mem, flags) != 0) {
        p->killed = 1;
        // refdec(pa);
        kfree(mem);
        goto end;
      }
    }
    end:
  }
  ...
}
```

## Fix `copyout()` function

### Objective

> When copying data from the kernel to a user virtual address, the destination address may correspond to a copy-on-write (COW) page. In this case, the COW page must be handled in a manner similar to the logic used in the `usertrap` function. If the page is identified as a COW page, a new physical page should be allocated and the contents of the original COW page should be copied into the newly allocated page. After the copy is completed, the faulting virtual page should be remapped to the new physical page so that the write operation can proceed without modifying the shared COW page.

### Implementation

The `copyout()` function has to be updated to handle the case where a page that is being copied is a COW page. We can reuse a lot of the code from `usertrap()`.

The implementation is as follows:

1. Set `va0` to the page-aligned starting virtual address of the page that contains the destination virtual address.
2. Traverse the page table and get the corresponding physical address; set `pa0` to the physical address. If `pa0` equals 0, return -1 (invalid address).
3. Traverse the page table to find the PTE for the virtual address. If no PTE is found, return -1 (error).
4. Check if the `PTE_COW` flag is set in the PTE.
5. Extract the privilege flags from the PTE.
6. Record the `PTE_W` privilege flag.
7. Clear the `PTE_COW` privilege flag.
8. Allocate a new page of physical memory using `kalloc()`. If `kalloc()` returns null, return -1 (error).
9. Using `memmove()`, copy the contents of the original page to the new page.
10. Decrease the reference count of the original physical page.
11. Unmap from the original page.
12. Map to the new page.
13. Update `pa0` to point to the new physical page.

#### `copyout()` (`kernel/vm.c`)

```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    pte_t *pte = walk(pagetable, va0, 0);
    if (!pte) {
      return -1;
    }
    if (*pte & PTE_COW) {
      uint flags = PTE_FLAGS(*pte);
      flags |= PTE_W;
      flags &= (~PTE_COW);
      char *mem = kalloc();
      if (!mem) {
        return -1;
      }
      memmove(mem, (void*)pa0, PGSIZE);
      refdec(pa0);
      uvmunmap(pagetable, va0, PGSIZE, 0);
      if (mappages(pagetable, va0, PGSIZE, (uint64)mem, flags) != 0) {
        // refdec(pa0);
        kfree(mem);
        return -1;
      }
      pa0 = (uint64)mem;
    }
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);
    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```
