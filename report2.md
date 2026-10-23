# LAB 2 – Report

Zhuchkova Sofia FAF-241

## Part 1

1. List three programs xv6 ships with.

Three programs included in xv6 are `ls`, `cat`, and `echo`.

2. Which two OS features must exist for a pipe to work?

A pipe requires process management and inter-process communication (IPC), allowing one process to send its output to another process as input.

Terminal output:
```
$ ls | grep c
cat            2 3 36760
echo           2 4 35624
wc             2 17 37760
sync           2 22 34976
console        3 23 0
```

3. In one sentence: how does the xv6 shell compare to the Linux shell from Lab 1?

The xv6 shell supports basic commands and pipes like the Linux shell, but it has fewer features because xv6 is a much simpler operating system.

## Part 2

1. Which system calls does `user/cat.c` use, and what does each one ask the kernel to do?

`cat.c` uses `read()` to read data, `write()` to output data, and `exit()` to terminate the program when an error occurs.

Terminal output:
```
// From user/user.h
int exit(int);
int write(int, const void *, int);
int read(int, void *, int);
```

2. In which kernel file, and on which line, is `sys_read` implemented?

`sys_read` is implemented in `kernel/sysfile.c` on line 69.

Terminal output:
```
vboxuser@Name:~/xv6-riscv$ grep -n "sys_read\|sys_write" kernel/sysfile.c | head
69:sys_read(void)
83:sys_write(void)
```

3. In one sentence: what is the difference between the code in `kernel/` and the code in `user/`?

The `kernel/` directory contains code that manages system resources and provides system calls, while `user/` contains programs that run outside the kernel and request its services.

## Part 3
user/sleep.c:
```
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[]) {
        if(argc != 2){
                fprintf(2, "usage: sleep <ticks>\n");
                exit(1);
        }
        pause(atoi(argv[1])); //system call into the k>
        exit(0);
}
```
