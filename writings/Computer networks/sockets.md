Its a endpoint where 2 process connect to communicate. a abstraction given by the os for 2 process to communicate. (maybe in the same machine or via a network)
A sockets is a combination of a protocal to transport data, a ip address and a port. 

The OS socket abstraction (the Berkeley sockets API: `socket()`, `bind()`, `connect()`, `send()`, `recv()`, etc.) was explicitly designed to be protocol-agnostic.
One uniform interface that can sit on top of many different protocol families: TCP, UDP, Unix domain, historically even IPX/AppleTalk, raw IP, etc.
The interface doesn't care what protocol is underneath you interact with a socket via the same handful of syscalls regardless of what's plugged in below it.
Where the coupling happens is at instantiation.
The moment you call `socket(family, type, protocol)`, the OS binds that specific socket instance to a specific protocol implementation in the kernel.
From then on, that instance's behavior (how `send()` behaves, whether `connect()` does a handshake, how the OS tracks its state) is entirely determined by the protocol you picked.

web sockets are the same? do they use UDP? how do they relate to this
what is a checksum
##### File descriptors
A file descriptor (FD) in C is a small integer that the operating system gives your program to represent an open resource (usually a file, socket, pipe, terminal, etc). Think of it as a handle to something the OS manages.
File descriptor 3 belongs to hello.txt, which this process opened.

###### Why not just use a pointer?
Because the file is not inside your program's memory.
Your program cannot directly access:

 ```
Disk
 └── hello.txt
 ```
 
The OS controls hardware access. The file descriptor is a safe reference:

```
Your program
      |
      | fd = 3
      ↓
Kernel
      |
      ↓
Disk file
```


| Function       | What it does                       | Allocates memory? | Safe size limit? |
| -------------- | ---------------------------------- | ----------------- | ---------------- |
| `snprintf()`   | Creates/formats a string           | No                | Yes              |
| `strcpy()`     | Copies a string                    | No                | No               |
| `strdup()`     | Copies a string to new heap memory | Yes               | N/A              |
| `strcmp()`<br> | Compares two strings               | No                | N/A              |

When you write:

```c
struct sockaddr_un addr;
```

you **create space for the struct**, but C does **not automatically initialize it**.

The memory is just whatever bits happened to already be there.

Example:

```c
int main() {
    int x;

    printf("%d\n", x);
}
```

`x` was never assigned a value, but it still prints something. Maybe:

```
32764
```

or:

```
-18273642
```

because the memory where `x` lives previously contained something else.

---

Same thing with structs:

```c
struct sockaddr_un addr;
```

might create memory like:

```
addr
+----------------+
| A7 39 FF 82    |  sun_family (garbage)
+----------------+
| 4B 91 22 00    |
| DE 44 8C 71    |
| ...            |  sun_path (garbage)
+----------------+
```

The OS did not give you a clean empty struct. C just reserved some bytes on the stack.

---

### Where did the garbage come from?

The stack is reused.

Imagine a function:

```c
void test() {
    char password[20];

    strcpy(password, "secret123");
}
```

After `test()` returns, that memory is available again.

Later:

```c
void other() {
    struct sockaddr_un addr;
}
```

The compiler might place `addr` in the same stack area.

The old bytes might still be there:

```
secret123
```

even though `addr` is a completely new variable.

C does not wipe memory for performance reasons.

---

### But what about the fields we set?

If you do:

```c
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/socket");
```

then those fields are initialized.

But structs can contain **padding bytes**.

Example:

```c
struct Example {
    char a;  // 1 byte
    int b;   // 4 bytes
};
```

The compiler might arrange it like:

```
+----+----+----+----+----+----+----+
| a  | pad| pad| pad| b           |
+----+----+----+----+----+----+----+
```

The padding exists so the CPU can access `b` efficiently.

You set:

```c
example.a = 'A';
example.b = 10;
```

but the padding bytes might still contain garbage.

---

### Why does `memset` help?

This:

```c
struct sockaddr_un addr;

memset(&addr, 0, sizeof(addr));
```

says:

> "Before I use this struct, make every byte inside it zero."

Now:

```
00 00 00 00 00 00 00 00 ...
```

Then you fill in the important fields:

```c
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/socket");
```

---

This is very different from languages like Java, Go, or Python where new objects usually get initialized automatically. In C, declaring a variable only gives you **memory**, not a clean object. You must initialize it yourself.
