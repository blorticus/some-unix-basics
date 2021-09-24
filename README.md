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

If you execute this, nothing is printed to the terminal (unless there is an error, which would be written to *stderr*).  Rather, *stdout* of the `cat` command is redirected to the file `/tmp/copy_of_interfaces.txt`.  There is a variant of the `>` operator, where a particular file descriptor from the left-hand operand is redirected.  For example:

```bash
rm /tmp/foo 2> /dev/null
```

Recall that for all processes, file descriptor "2" is *stderr* (unless it was moved by the process, but that is rare).  The command above would take any error output from `rm` and redirect it to the block device `/dev/null`.  `/dev/null` is a special device that accepts input and discards it.  That previous command, then, is a common pattern for discarding errors, supressing them from being printed to the terminal.

It is possible to have multiple redirectors, as in:

```bash
/usr/local/bin/myapp 2>$HOME/error.log >$HOME/output.log &
```

This would write *stderr* from `/usr/local/bin/myapp` to `$HOME/error.log` and *stdout* from the same process to `$HOME/output.log`.  This is a command pattern when one wishes to background a long-running process.  Notice the ampersand (`&`) at the end.  In a shell, when you type a command (like `/usr/local/bin/myapp`), it launches that application as a separate process (strictly speaking, the shell *forks* itself -- making a copy of itself and its environment -- then performs an *exec*, which replaces its command stream and static data with those found in the referenced executable).  Ordinarily, the sub-process is attached to the shell.  If you exit the shell, it will reap any attached sub-processes.  When the ampersand is provided, however, the process is detached from the parent process (the shell).  Exiting the shell has no effect on the process.  This is conventionally called *backgrounding* a process.

It is common to attach processes in a shell *pipeline*.  Consider the following example:

```bash
cat /etc/passwd | grep myself
```

`grep` searches an incoming byte stream for text matching that supplied ("myself", in this example).  Often, `grep` is provided a filename, and it searches that file for the match text.  However, if no filename is provided, it reads from its own *stdin*.  Recall that *stdin* is usually attached to keyboard input to the terminal.  However, if a process is on the right-hand side of a pipe (`|`), then its *stdin* is instead *stdout* of the command on the left side of the pipe.  As we've seen, `cat` reads from a file and prints the contents to its *stdout*.  Because of the pipe in the example command, the shell attaches *stdout* from `cat` to *stdin* of `grep` instead of the terminal.  Pipelines can chained, as in:

```bash
cat /etc/passwd | grep myself | tee $HOME/all_about_me.txt
```

The `tee` command takes input from its *stdin*, prints it to its *stdout* (just like `cat`), but also writes that same input stream to a file (`$HOME/all_about_me.txt`, in this example).

Consider the following:

```bash
cat /etc/passwd | less
```

`less` is a *pager*.  It reads a stream from a file, or if passed no file, from its *stdin*.  It presents the stream as a set of scrollable pages.

Hopefully by now, it is clear what this shell command set does.  `cat` reads the contents of `/etc/passwd` and prints them to its *stdout*.  Because of the pipe, that printed stream is redirected to *stdin* of `less`.  But what if the command prints some things to *stdout* but also has errors?  Those will be written to *stderr*.  Those error lines will not go to `less` if a simple pipe is used.  That's because the pipe connects *stdout* of the left-side to *stdin* of the right-side.  It doesn't change *stderr*, which goes to the terminal, by default.  However, we can redirect *stderr* of a process to its own *stdout*:

```bash
/usr/local/bin/myapp 2>&1 | less
```

The angle operator redirects *stderr*.  We already saw that.  Instead of sending it to a file, however, we used the *file descriptor duplication* operator.  That's the `&`.  In this context, it means "duplicate to the file descriptor that follows".  In other words `&1` means "duplicate to *stdout*".  So, `2>&1` means "duplicate *stderr* to *stdout*" and as a consequence, send anything from *stderr* to wherever *stdout* is pointing.  Since `myapp` is on the left of pipe, its *stdout* is redirected to *stdin* of `less`.  Because we duplicated *stderr* into *stdout* for `myapp`, now anything `myapp` prints to its *stderr* and *stdout* all goes to `less`.

## Commands

We will look briefly at the following commands:

1. `sort`
2. `unique`
3. `cut`
4. `sed`
5. `awk`

We will also look more closely at:
6. `grep`

### sort

The `sort` command does exactly what you'd expect: it reads a stream and sorts it, printing the sorted stream to stdout.  The sort function is applied line-by-line.  By default, the sort method is *lexical*, which means, sorted according to ASCII order (or UTF-8, depending on the version of `sort`).  Notice that this means that, given the following input:

```text
An egg
apples galore
Zelda
1
2
12
50
105
```

`sort` would produce the following output:

```text
An egg
Zelda
apples galore
1
105
12
2
50
```

That's because, in the ASCII character set, A - Z is before a - z, which is before 0 - 9.  For the numbers, the sort is purely lexical.  So anything starting with the character "0" will be before anything starting with the character "1", and so forth.

To compel `sort` to sort numerically, pass it the `-n` flag, as in:

```bash
cat <<EOF | sort -n
1
105
12
2
50
EOF
```

The `<<` is a *here-doc* operator.  We won't cover it further here, but for this example, know that it means everything between the line with `<<EOF` and the literal `EOF` is treated like a file stream.  Because of `cat <<EOF` that stream is redirected to *stdin* of the `cat` command, which then prints it to *stdout*.  That is, of course, captured by *stdin* of the `sort` command.  The command above would yield:

```text
1
2
12
50
102
```

If you use numeric sort, anything that isn't a number is sorted lexically.
