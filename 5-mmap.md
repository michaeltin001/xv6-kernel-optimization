# MMAP

## Objective

> The `mmap` and `munmap` system calls allow UNIX programs to exert detailed control over their address spaces. They can be used to share memory among processes, to map files into process address spaces, and as part of user-level page fault schemes such as garbage-collection algorithms.

## Implementation

The implementation initially requires us to add flags/definitions for the syscalls. To successfully compile the code without errors, we need to add “include guards” at the beginning of `file.h`, `fs.h`, and `sleeplock.h`.

Add `$U/_mmaptest\` in `Makefile` to add `mmaptest` test.

#### `Makefile`

```
+ $U/_mmaptest\
```

Add `entry("mmap"); entry("munmap");` in `user/usys.pl`.

#### `user/usys.pl`

```
+entry("mmap");
+entry("munmap");
```

Add the following function definitions in `user/user.h`.

#### `user/user.h`

```
+void* mmap (void*, unsigned int, int, int, int, unsigned int);
+int munmap (void*, unsigned int);
```

Add the following definitions in `kernel/syscall.h`.

#### `kernel/syscall.h`

```
+#define SYS_mmap 23
+#define SYS_munmap 24
```

Add the following mapping in `kernel/syscall.c`.

#### `kernel/syscall.c`

```
+[SYS_mmap] sys_mmap,
+[SYS_munmap] sys_munmap,
```

Add the following definitions in `kernel/syscall.c`.

#### `kernel/syscall.c`

```
+extern uint64 sys_mmap(void);
+extern uint64 sys_munmap(void);
```

Add the following definitions in `kernel/fcntl.h`.

#### `kernel/fcntl.h`

```
+#define PROT_READ 0x001
+#define PROT_WRITE 0x002
+#define MAP_PRIVATE 0x001
+#define MAP_SHARED 0x002
```

We need to define the VMA structure (recording start, length, permission, flags and file for each mapped memory range).

#### `kernel/proc.h`

```c
+struct vma {
+ uint64 addr;
+ int len;
+ int prot;
+ int flags;
+ struct file *f;
+ int offset;
+ int valid;
+};
```

We need to add a table in `struct proc` of all the VMAs for a process. Since the `Xv6` kernel doesn't have a memory allocator in the kernel, it's OK to declare a fixed size array of VMAs and allocate from that array as needed. A size of 16 should be sufficient.

#### `kernel/proc.h`

```c
+ struct vma vma_table[16];
```

We also need to initialize all members of the VMA struct for all VMAs in `allocproc()`.

#### `kernel/proc.c`

```c
+ for(int i = 0; i < 16; ++i) {
+   p->vma_table[i].addr = 0;
+   p->vma_table[i].len = 0;
+   p->vma_table[i].prot = 0;
+   p->vma_table[i].flags = 0;
+   p->vma_table[i].f = 0;
+   p->vma_table[i].offset = 0;
+   p->vma_table[i].valid = 0;
+ }
```

We can now complete the implementation of `mmap()`.

The implementation is as follows:

1. Validate the input of all arguments (`addr`, `len`, `prot`, `flags`, `f`, and `offset`). Return an error if any of the arguments are negative.
2. Return an error if the file is not readable and read permission is requested.
3. Return an error if the file is not writable and write permission is requested.
4. Iterate through the VMA table to find a free slot. Return an error if no free slot is found.
5. Iterate through the VMA table to find an unused region in which to map the file. If there is overlap, adjust the virtual address to be after the end of the VMA in use.
6. Initialize all fields of the chosen VMA entry (which was found in Step 4).
7. Increase the file’s reference count.
8. Return the starting virtual address of the mapped region.

#### `kernel/sysfile.c`

```c
+uint64
+sys_mmap(void)
+{
+ uint64 addr;
+ int len;
+ int prot;
+ int flags;
+ struct file *f;
+ int offset;
+
+ if (argaddr(0, &addr) < 0) {
+   return -1;
+ }
+ if (argint(1, &len) < 0) {
+   return -1;
+ }
+ if (argint(2, &prot) < 0) {
+   return -1;
+ }
+ if (argint(3, &flags) < 0) {
+   return -1;
+ }
+ if (argfd(4, 0, &f) < 0) {
+   return -1;
+ }
+ if (argint(5, &offset) < 0) {
+   return -1;
+ }
+
+ if (!(f->readable) && (prot & PROT_READ)) {
+   return -1;
+ }
+ if (!(f->writable) && (prot & PROT_WRITE) && !(flags & MAP_PRIVATE)) {
+   return -1;
+ }
+
+ struct proc *p = myproc();
+ struct vma *vma = 0;
+ for (int i = 0; i < 16; ++i) {
+   if ((p->vma_table[i].valid == 0)) { // found one
+     vma = &p->vma_table[i];
+     break;
+   }
+ }
+ if (vma == 0) {
+   return -1;
+ }
+
+ uint64 va = 0x40000000; // starting address 0x40000000
+ for (int i = 0; i < 16; ++i) {
+   if (p->vma_table[i].valid == 1) { // 1 = used
+     uint64 start_va = p->vma_table[i].addr;
+     uint64 end_va = (p->vma_table[i].addr + p->vma_table[i].len);
+     if ((va >= start_va) && (va < end_va)) {
+       va = PGROUNDUP(end_va);
+     }
+   }
+ }
+
+ vma->addr = va;
+ vma->len = len;
+ vma->prot = prot;
+ vma->flags = flags;
+ vma->f = f;
+ vma->offset = offset;
+ vma->valid = 1;
+ filedup(vma->f);
+
+ return va;
+}
```

We can now complete the implementation of `usertrap()` (similar to Lazy Page Allocation). The implementation is as follows:

1. Identify that a page fault has occurred (`r_scause() = 13` or `15`).
2. Obtain the faulting address.
3. Iterate through the VMA table and find the relevant VMA (the VMA that contains the faulting address). Kill the process if the VMA is not found.
4. Round down the faulting address to the nearest page boundary.
5. Calculate the offset within the mapped file that corresponds to the faulting address.
6. Allocate a new physical page using `kalloc()`. Kill the process if allocation fails.
7. Initialize the new physical page to zero.
8. If the page fault was a result of a load/store operation (`r_scause() = 15`) and the mapping is set to `MAP_PRIVATE`, skip the file read.
9. Read 4096 bytes of the relevant file into the page using `readi()`.
10. Set the appropriate flags for the PTE (`PTE_R?`, `PTE_W?` + `PTE_U`).
11. Map the allocated physical page into the user’s address space.

#### `kernel/trap.c`

```c
+ else if ((r_scause() == 13) || (r_scause() == 15)) {
+   uint64 va = r_stval();
+   struct vma *vma = 0;
+
+   for (int i = 0; i < 16; ++i) {
+     if (p->vma_table[i].valid == 0) {
+       continue;
+     }
+     uint64 start_va = p->vma_table[i].addr;
+     uint64 end_va = (p->vma_table[i].addr + p->vma_table[i].len);
+     if ((va >= start_va) && (va < end_va)) {
+       vma = &p->vma_table[i];
+       break;
+     }
+   }
+   if (vma == 0) {
+     p->killed = 1;
+     goto end;
+   }
+
+   uint64 fault_va = PGROUNDDOWN(va);
+   int offset = vma->offset + (fault_va - vma->addr);
+   pte_t *pte = walk(p->pagetable, fault_va, 0);
+ 
+   char *mem = kalloc();
+   if (mem == 0) {
+     p->killed = 1;
+     goto end;
+   }
+   memset(mem, 0, PGSIZE);
+
+   if ((r_scause() == 15) && (vma->flags & MAP_PRIVATE)) {
+     if (pte) {
+       goto skip;
+     }
+   }
+
+   struct file *f = vma->f;
+   begin_op(f->ip->dev);
+   ilock(f->ip);
+   readi(f->ip, 0, (uint64)mem, offset, PGSIZE);
+   iunlock(f->ip);
+   end_op(f->ip->dev);
+   int flags = PTE_U;
+   if (vma->prot & PROT_READ) {
+     flags |= PTE_R;
+   }
+   if (vma->prot & PROT_WRITE) {
+     if (!(vma->flags & MAP_PRIVATE)) {
+       flags |= PTE_W;
+     }
+   }
+
+ skip:
+   if (mappages(p->pagetable, fault_va, PGSIZE, (uint64)mem, PTE_W|PTE_R|PTE_U) != 0) {
+     p->killed = 1;
+     kfree(mem);
+     goto end;
+   }
+ end:
+ }
```

We can now complete the implementation of `munmap()`. The implementation is as follows:

1. Validate the input of all arguments (`addr`, `len`). Return an error if any of the arguments are negative.
2. Iterate through the VMA table and find the relevant VMA (the VMA that overlaps with the region being unmapped). Return an error if the VMA is not found.
3. If the VMA was mapped with the `MAP_SHARED` flag, write the contents of the unmapped page back to the mapped file.
4. Unmap the specific pages using `uvmunmap()`.
5. If `munmap` has removed all pages of a previous `mmap`, decrement the reference count of the corresponding `struct file`.
6. Return a success (0) if the function completes successfully.

#### `kernel/sysfile.c`

```c
+uint64
+sys_munmap(void)
+{
+ uint64 addr;
+ int len;
+
+ if (argaddr(0, &addr) < 0) {
+   return -1;
+ }
+ if (argint(1, &len) < 0) {
+   return -1;
+ }
+
+ struct proc *p = myproc();
+ struct vma *vma = 0;
+ for (int i = 0; i < 16; ++i) {
+   if (p->vma_table[i].valid == 0) {
+     continue;
+   }
+   uint64 start_va = p->vma_table[i].addr;
+   uint64 end_va = (p->vma_table[i].addr + p->vma_table[i].len);
+   if ((addr >= start_va) && (addr < end_va)) {
+     vma = &p->vma_table[i];
+     break;
+   }
+ }
+ if (vma == 0) {
+   return -1;
+ }
+
+ if (vma->flags & MAP_SHARED) {
+   for (uint64 va = addr; va < (addr + len); va += PGSIZE) {
+     pte_t *pte = walk(p->pagetable, va, 0);
+     if (pte) {
+       int offset = (va - (vma->addr + vma->offset));
+       begin_op(vma->f->ip->dev);
+       ilock(vma->f->ip);
+       writei(vma->f->ip, 1, addr, offset, len);
+       iunlock(vma->f->ip);
+       end_op(vma->f->ip->dev);
+     }
+   }
+ }
+
+ uvmunmap(p->pagetable, addr, len, 1);
+
+ if (addr == vma->addr) {
+   if (len == vma->len) {
+     fileclose(vma->f);
+     vma->valid = 0;
+   }
+ }
+
+ return 0;
+}
+
```

We also need to add some additional code to `exit()` and `fork()` in order to pass the entirety of `mmaptest`.

Modify `exit` to unmap the process's mapped regions as if `munmap` had been called.

#### `kernel/proc.c`

```c
+ for (int i = 0; i < 16; ++i) {
+   if (p->vma_table[i].valid == 1) {
+     uvmunmap(p->pagetable, p->vma_table[i].addr, p->vma_table[i].len, 1);
+     if (p->vma_table[i].f) {
+       fileclose(p->vma_table[i].f);
+     }
+     p->vma_table[i].valid = 0;
+   }
+ }
```

Modify `fork` to ensure that the child has the same mapped regions as the parent. Increment the reference count for a VMA's `struct file`.

#### `kernel/proc.c`

```c
+ for (int i = 0; i < 16; ++i) {
+   if (p->vma_table[i].valid == 1) {
+     np->vma_table[i] = p->vma_table[i];
+     if (p->vma_table[i].f) {
+       filedup(p->vma_table[i].f);
+     }
+   }
+ }
```

To avoid panics due to Lazy Allocation, we also implement modifications to `uvmcopy()` and `uvmunmap()`. In both functions, we simply comment out the panics and continue iterating through the loop.

#### `kernel/vm.c`

```c
+ if ((pte = walk(pagetable, a, 0)) == 0) {
+   // panic("uvmunmap: walk");
+   a += PGSIZE;
+   if (a > last) {
+     break;
+   }
+   continue;
+ }
+ if ((*pte & PTE_V) == 0) {
+   // printf("va=%p pte=%p\n", a, *pte);
+   // panic("uvmunmap: not mapped");
+   a += PGSIZE;
+   if (a > last) {
+     break;
+   }
+   continue;
+ }
```

#### `kernel/vm.c`

```c
+ if((pte = walk(old, i, 0)) == 0) {
+   // panic("uvmcopy: pte should exist");
+   continue;
+ }
+ if((*pte & PTE_V) == 0) {
+   // panic("uvmcopy: page not present");
+   continue;
+ }
```
