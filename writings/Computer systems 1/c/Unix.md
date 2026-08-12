---
Title: C vs POSIX
date: 2026-07-23
---
C is a programming language and the language itself defines things like:

```c
int x = 5;

if (x > 3) {
    printf("Hello\n");
}
```

It also defines a **standard library**, which includes functions like:

```c
printf()
scanf()
malloc()
free()
fopen()
fclose()
fread()
fwrite()
memcpy()
strlen()
```

These are specified by the ISO C standard and are intended to work on any platform with a C compiler (Linux, Windows, macOS, embedded systems, etc.).

---

## POSIX

**POSIX is an operating system standard.**

It defines a common API that Unix-like operating systems provide.

POSIX includes functions such as:

```c
open()
close()
read()
write()
socket()
bind()
listen()
accept()
fork()
exec()
pipe()
mmap()
```

These functions are **not part of the C language**. They are provided by the operating system.

You'll usually include headers like:

```c
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
```

These are POSIX headers.

---

## Example

### Standard C

```c
#include <stdio.h>

FILE *fp = fopen("test.txt", "r");

char buf[100];
fread(buf, 1, sizeof(buf), fp);

fclose(fp);
```

Everything here is standard C.

---

### POSIX

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("test.txt", O_RDONLY);

char buf[100];
read(fd, buf, sizeof(buf));

close(fd);
```

Everything here is POSIX.

---

## Why does POSIX exist?

Imagine writing software in the 1980s.

There were many Unix systems:

* SunOS
* AIX
* HP-UX
* BSD
* Solaris

Each had slightly different system calls.

POSIX standardized them so software could be portable across Unix systems.

Today, Linux and macOS implement most of POSIX, so the same code often works on both.

---

## Why do both APIs exist?

The C standard library was designed to work on many kinds of systems, even those without concepts like processes or sockets.

POSIX adds operating system features that C itself doesn't define.

For example:

| Task              | Standard C | POSIX                          |
| ----------------- | ---------- | ------------------------------ |
| Open file         | `fopen()`  | `open()`                       |
| Read file         | `fread()`  | `read()`                       |
| Write file        | `fwrite()` | `write()`                      |
| Memory allocation | `malloc()` | (uses C's `malloc()`)          |
| Print text        | `printf()` | `write()` can also write bytes |
| Create process    | ❌          | `fork()`                       |
| Execute program   | ❌          | `exec()`                       |
| Create socket     | ❌          | `socket()`                     |
| Accept connection | ❌          | `accept()`                     |

Notice that networking, processes, and sockets are **not part of C**. They come from POSIX (or platform-specific APIs on non-POSIX systems).

---

## Which should you use?

* If you're writing portable C that should compile almost anywhere, prefer the C standard library where possible.
* If you're writing software for Linux, macOS, or other Unix-like systems (servers, shells, networking, daemons, etc.), you'll use POSIX extensively.

Since you're building a Unix domain socket server, you're already using several POSIX functions:

```c
socket()
bind()
listen()
accept()
read()
write()
close()
unlink()
```

while still using standard C functions alongside them:

```c
printf()
malloc()
free()
memset()
strcpy()
```

This combination—standard C for language/runtime features and POSIX for operating system services—is how most C programs on Unix-like systems are written.
