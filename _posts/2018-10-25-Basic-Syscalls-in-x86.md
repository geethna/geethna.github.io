---
title: Basic Syscalls in x86
date: 2018-06-25 00:00:00 +0530
categories: [RE]
tags: [syscalls, x86]
---


### What is a Syscall?

System calls are requests that a program can pass to an operating system, to make it act on the program’s behalf in carrying out some task the program itself can’t do.

### Basic Syscalls:

- read
- write
- open
- close
- fork
- execve

Each syscall has its  own specific arguments.

### About read syscall:

We use this syscall when the program has to take an input from the user.

### Arguments for read:

`read(int file descriptor,void *buf,size_t count)`

- **file descriptor** – where to read the input (0-stdin, 1-stdout,2-stderr)
- **buffer**   – character array where the read content will be stored
- **count**    – number of bytes to read
- **return value**    – number of bytes that was read

### About write syscall:

Write syscall is used when you want the program to write out data from the buffer.

### Arguments for write:

`write(int file descriptor,void *buf,size_t count)`

- **file descriptor** – where to write the input (0-stdin, 1-stdout,2-stderror)
- ***buffer**   – address of the character array where the content to be written is stored
- **count**    – number of bytes to write
- **return value**    – number of bytes that was written

### About open syscall:

The open syscall is used to open a new file.

### Arguments for open:

`open(const char *path, int oflags)`

- ***path** – relative or absolute path to the file that is to be opened
- **oflags**  – A bitwise ‘or’ separated list of values that determine the method whether   it should be read only, read/write
- **return value** – Returns the file descriptor for the new file

### About close syscall:

The close syscall is used to close an open file descriptor.

### Arguments for close:

`close(int fildes)`

- **fildes**       – The file descriptor to be closed
- **return value** – Retuns a 0 upon success, and a -1 upon failure

### About exit syscall:

A computer process terminates its execution by making an exit system call.

### Arguments for exit:

`exit(int status)`

- **status**       – return code
- **return value** – it does not return any value

### About execve syscall:

It executes the program pointed to by filename.  This causes the program that is currently being run by the calling process to be replaced with a new program.

### Arguments for execve:

`execve(const char *filename, char *const argv[],char *const envp[]);`

- ***filename** – file that is to be opened
- **argv[]** – array of argument strings passed to the new program
 

- **envp[]** – array of argument strings passed as environment
 

- **return value** – does not return on success
 

Thank you!!