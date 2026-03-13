# XV6 Kernel Optimization

This project implements a collection of advanced system calls for the XV6 kernel. The goal is to expand on XV6's core functionality with file system traversal, inter-process communication, and optimized I/O operations. It is based on MIT's 6.1810 (formerly 6.828) Operating Systems course.

This project accomplishes the following objectives:

* Implements standard Unix user-level utilities: `sleep`, which pauses execution for a set number of ticks; `find`, which recursively searches directories for specific filenames; and `xargs`, which executes commands from standard input.
* Improves memory allocation by replacing statically declared arrays with a buddy allocator and implementing lazy page allocation for user-space heap memory.
* Optimizes the `fork()` system call by using a copy-on-write method that initially shares physical memory pages between parent and child processes, instead of duplicating them.
* Adds to the capabilities of the existing `Xv6` file system by supporting much larger file sizes and implementing symbolic links.
* Implements the `mmap` and `munmap` system calls, allowing processes to dynamically map files directly into memory and share memory with other processes.


|                    |                                                    |
|:-------------------|:---------------------------------------------------|
| Unix Utilities     | [1-unix-utilities.md](1-unix-utilities.md)         |
| Memory Allocation  | [2-memory-allocation.md](2-memory-allocation.md)   |
| Copy-on-Write Fork | [3-copy-on-write-fork.md](3-copy-on-write-fork.md) |
| File System        | [4-file-system.md](4-file-system.md)               |
| MMAP               | [5-mmap.md](5-mmap.md)                             |

