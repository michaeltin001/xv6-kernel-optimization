# Unix Utilities

## `sleep.c`

### Objective

> The goal of this exercise is to implement the UNIX `sleep` program for xv6. The program pauses execution for a user-specified number of ticks. A tick represents a unit of time defined by the xv6 kernel, specifically the interval between two interrupts from the timer chip. The implementation is placed in the file `user/sleep.c`. 

### Implementation

The implementation of the `sleep` program is straightforward.

1. Pass in the number of ticks as an argument (`argv[1]` in this case, as `argv[0]` is reserved for the command itself).
2. Convert `argv[1]`, which is a string, to an integer using `atoi()` function.
3. Call the `sleep` system call for the number of ticks passed in.

```c
int main(int argc, char *argv[]) {
  if (argc < 2) {
    fprintf(2, "Usage: sleep <number of ticks>\n");
    exit();
  }

  int ticks = atoi(argv[1]);

  if (ticks <= 0) {
    fprintf(2, "Invalid number of ticks\n");
    exit();
  }
  sleep(ticks);
  exit();
}
```

## `find.c`

### Objective

> The objective of this exercise is to implement a simplified version of the UNIX `find` program. The program searches through a directory tree and identifies all files that match a specified name. The implementation is placed in the file `user/find.c`. 

### Implementation

The code from the `ls` program can be reused for the `find` program because the functionality is very similar.

The purpose of the `ls` program is to list the contents of the directory where it is called from; it prints 1) all files and 2) all subdirectories and their contents. This is done through a single `switch` statement, where there is a case to print a file and a case to print a directory.

The case to print a directory inside the `switch` statement contains a `while` loop to explore all subdirectories. The `find` program needs to do this, but instead of printing all contents it prints only if a specific file name is found. This functionality is achieved by moving the `switch` statement to be inside of the `while` loop.

1. Pass in the path and the filename as arguments (`argv[1]` and `argv[2]`).
2. Check for errors at the beginning (cannot open or cannot stat).
3. Enter the `while` loop to begin reading the directory. If the directory name is `.` or `..`, do not explore and immediately `continue` the while loop.
4. Using `st.type`, determine the type of the object (`switch` statement).
   a. If it is a file, print it if it matches the passed in filename.
   b. If it is a directory, call the `find()` function again (recursion).

The goal is to recursively explore subdirectories until finding files; do not break the `while` loop until the type of the object is file, not directory.

```c
void find(char *path, char* filename) {
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;
  if((fd = open(path, 0)) < 0){
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }
  if(fstat(fd, &st) < 0){
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }
  strcpy(buf, path);
  p = buf+strlen(buf);
  *p++ = '/';
  while(read(fd, &de, sizeof(de)) == sizeof(de)){
    if(de.inum == 0)
      continue;
    // MT 1/18
    if ((strcmp(de.name, ".") == 0) || (strcmp(de.name, "..") == 0))
      continue;
    memmove(p, de.name, DIRSIZ);
    p[DIRSIZ] = 0;
    if(stat(buf, &st) < 0){
      printf("find: cannot stat %s\n", buf);
      continue;
    }
    // MT 1/18
    // printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);
    switch(st.type) {
      case T_FILE:
        // MT 1/18
        // printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);
        if (strcmp(de.name, filename) == 0) {
          printf("%s\n", buf);
        }
        break;
      case T_DIR:
        // MT 1/18
        // if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
        //   printf("find: path too long\n");
        //   break;
        // }
        find(buf, filename);
        break;
    }
  }
  close(fd);
}
```

## `xargs.c`

### Objective

> The goal of this exercise is to implement a simplified version of the UNIX `xargs` program. The program reads lines from standard input and executes a command for each line, passing the line as arguments to that command. The implementation is placed in the file `user/xargs.c`. 

### Implementation

The implementation of the `xargs` program is straightforward.

1. Declare an array of `char` pointers (`arrArgs`) which is size `MAXARG = 32`. This array stores the individual arguments as they are processed.
2. Declare a buffer array to store arguments (`buffer`) which is size `1024`.
3. The `for` loop reads the initial arguments (after `xargs`) and adds them to the argument pointers array.
4. After this, the remaining arguments are processed one character at a time. The characters are read into a temporary variable (`temp`) and then written to the buffer array (`buffer`).
5. When a space or newline is encountered, a null terminator (`'\0'`) is added to the buffer array (`buffer`). Then, the `char` pointer to the first letter in the string (`p`) is added to the array of char pointers (`arrArgs`).
6. When a newline is encountered, a child process is forked. The parent process then waits for the child process to finish (`wait` until `fork()`'s child exits).

```c
int main(int argc, char *argv[]) {
  if (argc < 2) {
    fprintf(2, "Usage: xargs <command>\n");
    exit();
  }
  char *arrArgs[MAXARG]; // 32
  char temp = '0';
  char buffer[1024];
  char *p = buffer;
  int position = 0;
  int numArgs = 0;
  for (int i = 1; i < argc; i++) {
    arrArgs[i - 1] = argv[i]; // Skip argv[0] which is xargs
    if (i == (argc - 1)) {
      numArgs = i;
    }
  }
  while (read(0, &temp, 1) != 0) {
    if ((temp == ' ') || (temp == '\n')) {
      buffer[position] = '\0';
      arrArgs[numArgs] = p;
      position++;
      numArgs++;
      if (temp == '\n') {
        p = buffer + position; // The space after the last arg
        if (fork() == 0) {
          exec(arrArgs[0], arrArgs);
        }
        else {
          wait();
        }
        numArgs = argc - 1;
      }
    }
    else {
      buffer[position] = temp;
      position++;
    }
  }
  exit();
}
```
