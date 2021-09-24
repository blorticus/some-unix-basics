# some-unix-basics

## Overview

This brief tutorial intends to share some basic, useful Unix commands.

## Redirection

Before introducing the commands, it is important to understand a few things.  Firstly, Linux has a concept of abstracted *I/O channels*.  A process can write to and/or read from these channels (some channels allow writing, some allow reading, and some allow both).  Naturally, this includes actual files on a filesystem, but it also includes things like network sockets and block devices.  When a process opens a handle to an *I/O channel*, the kernel assigns it an unsigned integer called a *file descriptor*.  The first file handle opened in a process gets file descriptor number 0, the second gets file descriptor number 1, and so forth.  This is true for each process, and the file descriptor 0 of process A is unrelated to file descriptor 0 of process B.  Each of those descriptors identify separate file handles.

When a process is launched, the kernel automatically opens the first three file descriptors for that process.  By convention, these are called *standard in* (which has file descriptor number 0), *standard out* (file descriptor 1) and *standard error* (file descriptor 2).  They are abbreviated *stdin*, *stdout* and *stderr*.  If a process is launched from a shell that is attached to a terminal, by default, the shell connects *stdin* to input from the terminal (usually, what you type on the keyboard), *stdout* to the terminal window and *stderr* to the terminal window.

Many Linux command-line tools write their normal output to *stdout* and error output to *stderr*.  Consider, for example, the `cat` command.  Ordinarily, one supplies a file name and `cat` prints the contents of the file to *stdout*.  Without any other redirection, that means the file contents are written to the terminal.  However, in most shells, *stdout* can in fact be redirected.  In most shells, the `>` operator is used to redirect *stdout* from the left-hand operand to a *file-like receiver* on the right side.  For example:

```bash
cat /etc/network/interfaces > /tmp/copy_of_interfaces.txt
```

If you execute this, nothing is printed to the terminal (unless there is an error, which would be written to *stderr*).  Rather, *stdout* of the `cat` command is redirected to the file `/tmp/copy_of_interfaces.txt`.  
