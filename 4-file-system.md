# File System

## Large Files

### Objective

> In its current form, Xv6 files are limited to 268 blocks, or `268*BSIZE` bytes, where `BSIZE` is 1024 bytes in Xv6. This limitation arises because each Xv6 inode contains 12 direct block numbers and one singly-indirect block number. The singly-indirect block refers to a block that can hold up to 256 additional block numbers, resulting in a total capacity of 12 + 256 = 268 blocks.
>
> To extend this limit, the Xv6 file system code should be modified to support a doubly-indirect block within each inode. This doubly-indirect block will contain 256 addresses of singly-indirect blocks, with each singly-indirect block containing up to 256 addresses of data blocks. With this structure, a file can consist of up to 65803 blocks, calculated as `256*256 + 256 + 11` blocks. The count will use 11 direct blocks instead of 12 because one of the direct block entries needs to store the address of the doubly-indirect block.

### Implementation

The implementation initially requires us to modify some constant definitions.

Change `#define NDIRECT` from `12` to `11`.

#### `NDIRECT` (`kernel/fs.h`)
```c
#define NDIRECT 11
```

Change `#define MAXFILE` accordingly.

#### `MAXFILE` (`kernel/fs.h`)
```c
#define MAXFILE (NDIRECT + NINDIRECT + (NINDIRECT * NINDIRECT))
```

Modify the declaration of `addrs[]` in `struct dinode` from `uint addrs[NDIRECT + 1]` to `[NDIRECT + 2]`.

#### `dinode{}` (`kernel/fs.h`)
```c
// On-disk inode structure
struct dinode {
  ...
  uint addrs[NDIRECT+2]; // Data block addresses
};
```

Modify the declaration of `addrs[]` in `struct inode` from `uint addrs[NDIRECT + 1]` to `[NDIRECT + 2]`.

#### `inode{}` (`kernel/file.h`)
```c
// in-memory copy of an inode
struct inode {
  ...
  uint addrs[NDIRECT+2];
};
```

We need to modify the `bmap()` function to support the doubly-indirect block. The `bmap()` function is responsible for translating a logical block number within a file to a physical block number on the disk.

The implementation is as follows:

1. Adjust the block number `bn` to account for the blocks already addressed by the singly-indirect block.
2. Check if the requested block number falls within the range addressable by the doubly-indirect block.
3. Check if the doubly-indirect block has been allocated; if not, allocate a new block using `balloc()`.
4. Read the doubly-indirect block from the disk into `bp` (buffer); then, cast the data into `a`.
5. Check if the respective singly-indirect block has been allocated; if not, allocate a new block using `balloc()`.
6. Release `bp` (buffer) for the doubly-indirect block.
7. Read the singly-indirect block from the disk into `bp` (buffer); then, cast the data into `a`.
8. Check if the respective direct block has been allocated; if not, allocate a new block using `balloc()`.
9. Release `bp` (buffer) for the singly-indirect block.
10. Return the address of the final data block.

#### `bmap()` (`kernel/fs.c`)

```c
// Return the disk block address of the nth block in inode ip.
// If there is no such block, bmap allocates one.
static uint
bmap(struct inode *ip, uint bn)
{
    ...
    bn -= NINDIRECT;
    if (bn < (NINDIRECT * NINDIRECT)) {
        if ((addr = ip->addrs[NDIRECT + 1]) == 0) {
            ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
        }
        bp = bread(ip->dev, addr);
        a = (uint*)bp->data;
        if ((addr = a[bn / NINDIRECT]) == 0) {
            a[bn / NINDIRECT] = addr = balloc(ip->dev);
            log_write(bp);
        }
        brelse(bp);
        bp = bread(ip->dev, addr);
        a = (uint*)bp->data;
        if ((addr = a[bn % NINDIRECT]) == 0) {
            a[bn % NINDIRECT] = addr = balloc(ip->dev);
            log_write(bp);
        }
        brelse(bp);
        return addr;
    }
    panic("bmap: out of range");
}
```

We also need to modify `itrunc()` accordingly to free all blocks of a file including doubly-indirect blocks. The `itrunc()` function is used to truncate a file, which effectively discards its contents.

The implementation is as follows:

1. Add a new `if` statement to handle the freeing of the doubly-indirect block.
2. Read the doubly-indirect block from the disk into `bp` (buffer); then, cast the data into `a`.
3. Iterate through the doubly-indirect block (outer loop). At each iteration…

   * Read the singly-indirect block from the disk into `bpi` (buffer); then, cast the data into `b`.
   * Iterate through the singly-indirect block (inner loop) to free all the data blocks using `bfree()`.
   * Release `bpi` (buffer) for the singly-indirect block.
   * Free the singly-indirect block itself using `bfree()`.
4. After the outer loop is complete…

   * Release `bp` (buffer) for the doubly-indirect block.
   * Free the doubly-indirect block itself using `bfree()`.
5. Clear the entry for the doubly-indirect block in the inode.

#### `itrunc()` (`kernel/fs.c`)

```c
// Truncate inode (discard contents).
// Only called when the inode has no links
// to it (no directory entries referring to it)
// and has no in-memory reference to it (is
// not an open file or current directory).
static void
itrunc(struct inode *ip) {
    int i, j, k;
    struct buf *bp, *bpi;
    uint *a, *b;
    ...
    if (ip->addrs[NDIRECT + 1]) {
        bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
        a = (uint*)bp->data;
        for (j = 0; j < NINDIRECT; j++) {
            if (a[j]) {
                bpi = bread(ip->dev, a[j]);
                b = (uint*)bpi->data;
                for (k = 0; k < NINDIRECT; k++) {
                    if (b[k]) {
                        bfree(ip->dev, b[k]);
                    }
                }
                brelse(bpi);
                bfree(ip->dev, a[j]);
            }
        }
        brelse(bp);
        bfree(ip->dev, ip->addrs[NDIRECT + 1]);
        ip->addrs[NDIRECT + 1] = 0;
    }
    ip->size = 0;
    iupdate(ip);
}
```

---

## Symbolic Links

### Objective

> Symbolic links, also known as soft links, refer to a linked file by its pathname. When a symbolic link is opened, the kernel follows the link and resolves it to the file it references. Symbolic links are similar to hard links, but there are important differences between them. Hard links are restricted to pointing to files on the same disk, whereas symbolic links can reference files across different disk devices.

### Implementation

We first need to implement the `sys_symlink()` system call.

The implementation is as follows:

1. Declare arrays to store the target and path strings. Retrieve the target and path arguments from the system call (return -1 if there is an error with this).
2. Begin the file system transaction.
3. Create a new inode for the symbolic link and check if the creation was successful (return -1 otherwise).
4. Calculate the length of the target path string.
5. Write the length of the target path to the inode.
6. Write the target path string to the inode and finish it with a null terminator.
7. Copy the modified in-memory inode to disk.
8. Unlock and drop the reference to the inode.
9. End the file system transaction.

#### `sys_symlink()` (`kernel/sysfile.c`)

```c
uint64
sys_symlink(void)
{
    //your implementation goes here
    char target[MAXPATH], path[MAXPATH];
    if ((argstr(0, target, MAXPATH) < 0) || (argstr(1, path, MAXPATH) < 0)) {
        return -1;
    }
    begin_op(ROOTDEV);
    struct inode *ip = create(path, T_SYMLINK, 0, 0);
    if (ip == 0) {
        end_op(ROOTDEV);
        return -1;
    }
    // writei(ip, 0, (uint64)target, 0, (strlen(target) + 1));
    int len = strlen(target);
    writei(ip, 0, (uint64)&len, 0, sizeof(int));
    writei(ip, 0, (uint64)target, sizeof(int), len + 1);

    iupdate(ip);
    iunlockput(ip);
    end_op(ROOTDEV);
    return 0;
}
```

We also need to modify the `sys_open()` system call to support symbolic links. To accomplish this, we will add a new `if` statement to the system call.

The implementation is as follows:

* Check if the inode represents a symbolic link, and that the `O_NOFOLLOW` flag is not set.
* Initialize the `depth` variable to `0`.
* Move into the `while` loop.

  * At the beginning of each iteration, check if `depth` exceeds `10`; if it does, we have encountered a cycle. In this case, we release the inode, end the operation, and return `-1` as an error code.
  * Read the length of the target path from the inode.
  * Read the target path string itself, including the null terminator at the end.
  * Unlock and drop the reference to the inode.
  * Resolve the path name and lock the inode.
  * Increment `depth`.

#### `sys_open()` (`kernel/sysfile.c`)

```c
uint64
sys_open(void)
{
    char target[MAXPATH], path[MAXPATH];
    ...
    if ((ip->type == T_SYMLINK) && ((omode & O_NOFOLLOW) == 0)) {
        int depth = 0;
        while ((ip->type == T_SYMLINK) && ((omode & O_NOFOLLOW) == 0)) {
            if (depth > 10) {
                iunlockput(ip);
                end_op(ROOTDEV);
                return -1;
            }
            int len = 0;
            readi(ip, 0, (uint64)&len, 0, sizeof(int));
            readi(ip, 0, (uint64)target, sizeof(int), len + 1);
            iunlockput(ip);
            ip = namei(target);
            if (ip == 0) {
                end_op(ROOTDEV);
                return -1;
            }
            ilock(ip);
            depth++;
        }
    }
    ...
    iunlock(ip);
    end_op(ROOTDEV);
    return fd;
}
```
