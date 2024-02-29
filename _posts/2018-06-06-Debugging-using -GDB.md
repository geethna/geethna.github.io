---
title: Debugging using GDB
date: 2018-06-06 00:00:00 +0530
categories: [RE]
tags: [re]
---

So here I am, up with my first blog post. This post is basically an intro on how to debug a program using gdb.

GDB or GNU Debugger is extensively used in debugging programs written using mainly Ada, C, C++, Objective-C, Free Pascal, Fortran, Go, Java. In this post I am going to explain how to get started with gdb.

## Prerequisites:

A basic knowledge about x32 assembly and C language.

## Installation:

First to check whether gdb is installed in you system or not try this command in you terminal


```
gdb --version
```

If that doesn’t work then you can probably check out this [link](https://www.gdbtutorial.com/tutorial/how-install-gdb). It explains how to install gdb very clearly.

### Getting Started:

Now the next task is to compile the program and disassemble it using gdb.

Command for compiling the program:

```
gcc -g -o program program.c
```

Command to open the program in gdb :

```
gdb executable
```

### Useful commands while using gdb:

`disas main` : to disassemble the main of the program

`set disassembly flavor-intel` : to change the disassembly to intel syntax

`info registers (i r)` : to view the current status of registers

`help (h)` : to get help on gdb commands

`b (break) <address>` : to set a breakpoint at the specified address

`r (run)`  : run the program until it encounters a breakpoint or to end 

`s (step)` : single step to descend into the functions

`n (next)` : single step without descending into the functions

`c (continue)` : continue till the next breakpoint or end

`l (list)` : to list the codes around the current point

`i b (info b)` : list all the breakpoints

`p (print) <address/variable>` : to print the value at the specified address or variable.

`p/x <variable>` : prints the value at the given address in hex form

`delete <number>` : to delete all breakpoints or the specified one

`dis <number>`  : disable the specified breakpoint

`en  <number>`  : to enable the specified breakpoint

`clear <function>` : to clear all breakpoints set in that function


### GDB-PEDA

PEDA – Python Exploit Development Assistance for GDB.

PEDA is a Python GDB script with many handy commands and a user friendly display. It helps in speeding up the exploit development process.

### Installation :

```
git clone https://github.com/longld/peda.git ~/peda
echo "source ~/peda/peda.py" >> ~/.gdbinit
```

### Useful commands while using gdb-peda :

`pd <function>` : to print the disasembly of the specified function

`b *(function/address)` : to set a breakpoint at the specified address or function

`r` : to run the program till the breakpoint or end

`ni` : single step to without descending into any functions

`si` : single step to descend into the functions

`help (h)` : to get help on gdb commands

`c (continue)` : continue till the next breakpoint or end

`delete` : to delete all breakpoints or the specified one

`print <$register/address/variable>` : to print the value at the specified address or variable.

`x/s <$register/address>` : to print the string at the given address 

[s in the above command can be replaced with x (hex), wx (integer) as preferred]
 

So these were some basic commands which will help you in getting started with gdb or gdb-peda. I hope you will find this article useful. If you have any questions or queries  feel free to contact me. Happy debugging 😉 !!

Thank you!!
