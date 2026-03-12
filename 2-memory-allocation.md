# Memory Allocation

## Heap Allocator

### Objective

> xv6 has only a page allocator and cannot dynamically allocate objects smaller
than a page. To work around this limitation, xv6 declares objects smaller than a
page statically. For example, xv6 declares an array of file structs, an array of
proc structures, and so on. As a result, the number of files the system can have
open is limited by the size of the statically declared file array, which has NFILE
entries (see kernel/file.c and kernel/param.h).
>
> The solution is to adopt the buddy allocator, which we have added to xv6 in
`kernel/buddy.c` and `kernel/list.c`. In `kernel/file.c`, the number of file structures should be limited by available
memory rather than `NFILE`.

### Implementation

The implementation of the Heap Allocator is straightforward and simple. The
main task is to revise the implementation of `filealloc()`.

The original code 1) acquires the lock protecting the file table, 2) loops through the
array to find a free file structure, 3) once found, sets the reference count of the
free file structure to 1, 4) returns a pointer to the allocated file structure and
releases the lock.

The new code 1) uses `bd_malloc()` to allocate a block of memory large enough to
hold a `file`, 2) acquires the lock protecting the file table, 3) sets the reference
count of the new `file` structure to 1, 4) releases the lock and returns a pointer to
the allocated file structure.

Additionally, we add `bd_free()` to the `fileclose` function to free memory that was
allocated by `bd_malloc()`.

With the implementation described here, `alloctest` successfully passes all tests.

#### `filealloc()` (`kernel/file.c`)

```c
// Allocate a file structure.
struct file*
filealloc(void)
{
    struct file *f;
    f = (struct file *) bd_malloc(sizeof(struct file));
    acquire(&ftable.lock);
    // for(f = ftable.file; f < ftable.file + NFILE; f++){
    //   if(f->ref == 0){
    //     f->ref = 1;
    //     release(&ftable.lock);
    //     return f;
    //   }
    // }
    f->ref = 1;
    release(&ftable.lock);
    return f;
}
```

#### `fileclose()` (`kernel/file.c`)

```c
// Close file f. (Decrement ref count, close when reaches 0.)
void
fileclose(struct file *f)
{
    struct file ff;
    acquire(&ftable.lock);
    if(f->ref < 1)
        panic("fileclose");
    if(--f->ref > 0){
        release(&ftable.lock);
        return;
    }
    ff = *f;
    f->ref = 0;
    f->type = FD_NONE;
    release(&ftable.lock);
    if(ff.type == FD_PIPE){
        pipeclose(ff.pipe, ff.writable);
    } else if(ff.type == FD_INODE || ff.type == FD_DEVICE){
        begin_op(ff.ip->dev);
        iput(ff.ip);
        end_op(ff.ip->dev);
    }
    bd_free((char *) f);
}
```

## Lazy Page Allocation

### Objective

> One of the many neat tricks an O/S can play with page table hardware is lazy
allocation of user-space heap memory. Xv6 applications ask the kernel for heap
memory using the `sbrk()` system call. In the kernel we've given you, `sbrk()`
allocates physical memory and maps it into the process's virtual address space.
However, there are programs that use `sbrk()` to ask for large amounts of memory
but never use most of it, for example to implement large sparse arrays. To
optimize for this case, sophisticated kernels allocate user memory lazily. That is,
`sbrk()` doesn't allocate physical memory, but just remembers which addresses are
allocated. When the process first tries to use any given page of memory, the CPU
generates a page fault, which the kernel handles by allocating physical memory,
zeroing it, and mapping it.

### Implementation

The implementation of the Lazy Page Allocation is complex and involves
modifications to multiple files.

Initially, we delete page allocation from the `sys_sbrk()` system call
implementation. The new `sys_sbrk()` simply increments the process’s size
(`myproc()->sz`) by `n` and returns the old size, without allocating memory.

#### `sys_sbrk()` (`kernel/sysproc.c`)

```c
uint64
sys_sbrk(void)
{
    int addr;
    int n;
    if(argint(0, &n) < 0)
        return -1;
    addr = myproc()->sz;
    // if(growproc(n) < 0)
    //   return -1;
    myproc()->sz += n; // Increment the process's size by n
    return addr;
}
```

We get the following result when attempting to run `echo hi`.

```
virtio disk init 0
hart 2 starting
hart 1 starting
init: starting sh
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
 sepc=0x000000000000124e stval=0x0000000000004008
va=0x0000000000004000 pte=0x0000000000000000
panic: uvmunmap: not mapped
```

We first have to fix `usertrap()` in `trap.c` so that `echo hi` can run in the shell
again. The code should respond to a page fault from user space by mapping a
newly-allocated page of physical memory at the faulting address, and then
returning back to user space to let the process continue executing.

The implementation of the modified `usertrap()` function is straightforward. We
simply modify the `else` branch in the `usertrap()` function. This is the core idea of the Lazy Page Allocation.

1. Check whether a fault is a page fault (`r_scause() == 13` or `15`)
2. Check whether the address is out of bounds; if it is, kill the process.
3. Allocate a new page using `kalloc()`, and map the address to the new page
   using `mappages()`; if either of these operations fail, kill the process.

#### `usertrap()` (`kernel/trap.c`)

```c
void
usertrap(void)
{
    ...
    else {
        if ((r_scause() == 13) || (r_scause() == 15)) {
            if ((r_stval() >= p->sz) || (r_stval() <= p->ustack)) {
                p->killed = 1;
                goto end;
            }
            uint64 a = PGROUNDDOWN(r_stval());
            char *mem = kalloc();
            if (mem == 0) {
                p->killed = 1;
                goto end;
            }
            memset(mem, 0, PGSIZE);
            if(mappages(p->pagetable, a, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0) {
                p->killed = 1;
                kfree(mem);
                // uvmdealloc(pagetable, a, oldsz);
                goto end;
            }
        }
        ...
    end:
    }
    ...
}
```

To check whether the address is out of bounds, we add `uint64 ustack` to the
`proc` struct in `proc.h`. This is the bottom of the user stack.

#### `proc{}` (`kernel/proc.h`)

```c
struct proc {
    ...
    // these are private to the process, so p->lock need not be held.
    uint64 kstack; // Bottom of kernel stack for this process
    uint64 ustack; // Bottom of user stack
    ...
};
```

#### `fork()` (`kernel/proc.c`)

```c
int
fork(void)
{
    ...
    np->ustack = p->ustack;
    np->parent = p;
    ...
}
```

#### `exec()` (`kernel/exec.c`)

```c
int
exec(char *path, char **argv)
{
    ...
    stackbase = sp - PGSIZE;
    p->ustack = stackbase;
    ...
}
```

This is not the expected behavior of XV6, so the OS will panic. We need to modify
functions in `vm.c` so that we can avoid this.

#### `walk()` (`kernel/vm.c`)

Do not panic on `MAXVA` check; simply comment this out.

```c
static pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
    // if(va >= MAXVA)
    //   panic("walk"); // change this
    ...
}
```

#### `mappages()` (`kernel/vm.c`)

Do not panic on remap condition; simply move to the next page.

```c
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
    ...
    if(*pte & PTE_V) {
        // panic("remap");
        a += PGSIZE;
        if (a > last) {
            break;
        }
        continue;
    }
    ...
}
```

#### `uvmunmap()` (`kernel/vm.c`)

Do not panic if a page does not have a PTE or if its VA is not mapped; simply move
to the next page.

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 size, int do_free)
{
    ...
    for(;;){
        if((pte = walk(pagetable, a, 0)) == 0) {
            // panic("uvmunmap: walk");
            a += PGSIZE;
            if (a > last) {
                break;
            }
            continue;
        }
        if((*pte & PTE_V) == 0) {
            // printf("va=%p pte=%p\n", a, *pte);
            // panic("uvmunmap: not mapped");
            a += PGSIZE;
            if (a > last) {
                break;
            }
            continue;
        }
        ...
    }
}
```

#### `uvmcopy()` (`kernel/vm.c`)

Do not panic if PTE does not exist or page is not present; simply continue loop.

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
    ...
    for(i = 0; i < sz; i += PGSIZE){
        if((pte = walk(old, i, 0)) == 0)
            // panic("uvmcopy: pte should exist");
            continue;
        if((*pte & PTE_V) == 0)
            // panic("uvmcopy: page not present");
            continue;
        ...
    }
}
```

We also comment out a `kalloc()` panic in the `sys_exec()` function.

#### `sys_exec()` (`kernel/sysfile.c`)

```c
uint64
sys_exec(void)
{
    ...
    // if(argv[i] == 0)
    //   panic("sys_exec kalloc");
    ...
}
```

Additionally, we implement a check in `sys_sbrk()` to handle negative arguments.

#### `sys_sbrk()` (`kernel/sysproc.c`)

```c
uint64
sys_sbrk(void)
{
    int addr;
    int n;
    if(argint(0, &n) < 0)
        return -1;
    addr = myproc()->sz;
    // if(growproc(n) < 0)
    //   return -1;
    myproc()->sz += n; // Increment the process's size by n
    if (n < 0) {
        uvmdealloc(myproc()->pagetable, addr, myproc()->sz);
        // uvmunmap(myproc()->pagetable, addr + n, addr - (addr + n), 1);
    }
    return addr;
}
```

The final modification we need to make is to handle the case in which a process
passes a valid address from `sbrk()` to a system call such as `read` or `write`, but the
memory for that address has not yet been allocated.

To handle this, we modify the `copyout()`, `copyin()`, and `copyinstr()` functions
which can be found in `vm.c`. These functions handle read and write operations
between user space and kernel space. We can use the Lazy Page Allocation that
was implemented in `usertrap()` earlier.

In each of the three functions, there is a specific section of code.

```c
pa0 = walkaddr(pagetable, va0);
if(pa0 == 0)
    return -1;
```

The function of this code is to check if the virtual address is mapped to a physical
address in the page table; else, it returns `-1`, indicating an error. This function
would be correct in the original implementation of XV6. However, because we are
using Lazy Page Allocation, we actually want to allocate memory when we come
across a virtual address that isn’t mapped to a physical address. Hence, we
modify the implementation of that code accordingly.

1. Check if the virtual address is mapped to a physical address.
2. If it isn’t, allocate memory using `kalloc()`, and map the address to the new
   page using `mappages()`; if either of these operations fail, return `-1` to
   indicate an error.
3. Check if the virtual address is mapped to a physical address again. If it
   somehow isn’t at this point, return `-1` to indicate an error.

Another subtle modification we make is that we stop casting the result of
`PGROUNDDOWN(srcva)` to type `uint`, because unsigned integers aren’t big enough to
store the result (also, `va0` is of type `uint64`, so this does not make sense
regardless).

**Old**

```c
va0 = (uint)PGROUNDDOWN(srcva);
```

**New**

```c
va0 = PGROUNDDOWN(srcva);
```

#### `copyout()` (`kernel/vm.c`)

```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
    uint64 n, va0, pa0;
    while(len > 0){
        va0 = PGROUNDDOWN(dstva);
        pa0 = walkaddr(pagetable, va0);
        if(pa0 == 0) {
            char *mem = kalloc();
            if(mem == 0)
                return -1;
            memset(mem, 0, PGSIZE);
            if(mappages(pagetable, va0, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U)!= 0){
                kfree(mem);
                return -1;
            }
        }
        pa0 = walkaddr(pagetable, va0);
        if(pa0 == 0)
            return -1;
        ...
    }
}
```

#### `copyin()` (`kernel/vm.c`)

```c
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
    uint64 n, va0, pa0;
    while(len > 0){
        va0 = PGROUNDDOWN(srcva);
        pa0 = walkaddr(pagetable, va0);
        if(pa0 == 0) {
            char *mem = kalloc();
            if(mem == 0)
                return -1;
            memset(mem, 0, PGSIZE);
            if(mappages(pagetable, va0, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U)!= 0){
                kfree(mem);
                return -1;
            }
        }
        pa0 = walkaddr(pagetable, va0);
        if(pa0 == 0)
            return -1;
        ...
    }
}
```

#### `copyinstr()` (`kernel/vm.c`)

```c
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
    uint64 n, va0, pa0;
    int got_null = 0;
    while(got_null == 0 && max > 0){
        va0 = PGROUNDDOWN(srcva);
        pa0 = walkaddr(pagetable, va0);
        if(pa0 == 0) {
            char *mem = kalloc();
            if(mem == 0)
                return -1;
            memset(mem, 0, PGSIZE);
            if(mappages(pagetable, va0, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U)!= 0){
                kfree(mem);
                return -1;
            }
        }
        pa0 = walkaddr(pagetable, va0);
        if(pa0 == 0)
            return -1;
        ...
    }
}
```
