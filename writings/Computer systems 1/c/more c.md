1
Core memory/process concepts in C 
##### Process Creation
- **`fork()`** – creates a child process, duplicating the parent's address space (copy-on-write in practice). Returns 0 in child, child's PID in parent, -1 on error.
- **`exec()` family** (`execl`, `execle`, `execlp`, `execv`, `execve`, `execvp`) – replaces the current process image with a new program. Doesn't create a new process.
- **`vfork()`** – older, faster variant of fork that shares the parent's memory until exec/exit (mostly deprecated now, but shows up in older code/exams).
- **PID / PPID** – `getpid()`, `getppid()` to get process and parent process IDs (ProcessID).
##### Process Termination
- **`exit()`** – terminates process, flushes stdio buffers, runs `atexit()` handlers.
- **`_exit()`** – terminates immediately, no cleanup, no buffer flushing. Used right after fork in the child in many patterns.
> what are the fork patterns? 
- **Exit status** – the value passed to exit/`_exit`, retrievable by the parent via wait.
> need to explain more this
- **Zombie process** – a terminated child whose exit status hasn't been reaped by the parent yet.
> why do we need the exit status, what is the exit status, is it a value in memory?, a file? wtf?? i need a full explainaiton from the start.
- **Orphan process** – a child whose parent terminated before it did (gets reparented, usually to init/PID 1).
> We need more detailed explainations on process and stuff
##### Waiting / Synchronization
- **`wait()`** – blocks until any child terminates, returns its PID and status.
> so like golangs/ waitgroups?
- **`waitpid()`** – waits for a specific child (or with flags like `WNOHANG` for non-blocking).
- **`WIFEXITED`, `WEXITSTATUS`, `WIFSIGNALED`, `WTERMSIG`** – macros to interpret the status returned by wait/waitpid.

- **File descriptor inheritance** – child inherits open fds from parent after `fork()` (crucial for how pipes actually work across processes).

i also want details low level from the start explaintions that starts from one corner and gradually explains all of these

## Inter-Process Communication (IPC)
- **Pipes** – `pipe()` creates an unnamed, unidirectional pipe (two fds: read end, write end). Only works between related processes (e.g., after fork).
- **Named pipes (FIFOs)** – `mkfifo()`, allow unrelated processes to communicate via a filesystem path.
- **Dup / dup2** – `dup()`, `dup2()` used to redirect file descriptors (classic pattern for wiring pipe ends to stdin/stdout).
- **Shared memory** – `shmget()`, `shmat()`, `shmdt()`, `shmctl()` (System V) or `mmap()` with `MAP_SHARED` (POSIX/BSD style).
- **Message queues** – `msgget()`, `msgsnd()`, `msgrcv()`.
- **Semaphores** – `sem_init()`, `sem_wait()`, `sem_post()` (POSIX) or `semget()`/`semop()` (System V) — used for synchronization, not data transfer.
- **Sockets** – for IPC across processes/machines (`socketpair()` for local IPC specifically).
## Signals
- **`signal()` / `sigaction()`** – register a handler for a signal.
- **`kill()`** – send a signal to a process.
- **`raise()`** – send a signal to yourself.
- **Common signals** – `SIGCHLD` (child terminated), `SIGKILL`, `SIGTERM`, `SIGINT`, `SIGSEGV`, `SIGPIPE` (writing to a pipe with no reader).
- **`sigprocmask()`** – block/unblock signals.
## Dynamic Memory Management
- **`malloc()` / `calloc()` / `realloc()` / `free()`** – heap allocation basics.
- **Memory leaks** – forgetting to `free()`.
- **Dangling pointers** – using memory after it's freed.
- **Double free** – freeing the same pointer twice.
- **`brk()` / `sbrk()`** – low-level system calls that adjust the heap's program break (what malloc uses under the hood).
- **`mmap()` / `munmap()`** – map memory directly (files or anonymous memory), often used for large allocations instead of the heap.
## Process Memory Layout
- **Text segment** – compiled code.
- **Data segment** – initialized globals/statics.
- **BSS segment** – uninitialized globals/statics.
- **Heap** – dynamically allocated memory, grows upward.
- **Stack** – local variables, function call frames, grows downward.
- **Address space / virtual memory** – each process has its own, mapped by the OS/MMU.
## Related Odds and Ends
- **`system()`** – runs a shell command (internally forks/execs).
- **Race conditions** – concurrent access issues between parent/child, often the whole point of `wait()`.
- **Copy-on-write (COW)** – how modern `fork()` avoids actually copying all memory immediately.

# Processes in C — From the Kernel Up

Forget C for a second. To the kernel, a process is just an entry in a table — on Linux it's a `struct task_struct`, sitting in kernel memory, that bundles:

- a **PID** (integer)
- a pointer to an **address space** (page tables — what memory this process can see)
- a **file descriptor table** (array of pointers to open files)
- saved **CPU register state** (so it can be paused and resumed)
- a **PPID** (who made it)
- an **exit status slot** (empty until it dies — see §8)

Every syscall in this guide (`fork`, `exec`, `exit`, `wait`) is just a well-defined operation on this struct. Nothing here is exotic — it's all edits to a table entry.

---

## 1. `fork()` — what literally happens

```c
pid_t pid = fork();
```

Step by step, on the kernel side:

1. Kernel allocates a **new** `task_struct` for the child.
2. Copies the parent's file descriptor table (the array — not the underlying files, see §13).
3. Copies the parent's page tables, marking all pages **copy-on-write** (COW) — no actual RAM is duplicated yet. If either process writes to a shared page later, the kernel faults, duplicates *that one page*, and lets them diverge.
4. Assigns the child a fresh PID.
5. Puts the child on the scheduler's run queue.
6. **Both processes now resume execution at the exact same point** — right after the `fork()` call — because the child's copied register state has the same program counter as the parent had.

That last point is the whole trick behind "`fork()` returns twice." It's not magic — there are just two `task_struct`s now, each independently resuming from the same saved instruction pointer. The return value is how each process tells itself apart:

| Who                | Return value                 |
| ------------------ | ---------------------------- |
| Parent             | child's PID (a positive int) |
| Child              | `0`                          |
| Either, on failure | `-1`, no child created       |

Why `0` for the child and not its own PID? Because the child doesn't need `fork()` to tell it its PID — it can call `getpid()` any time. The parent, on the other hand, has no way to learn the child's PID except through the return value. `0` is just a sentinel meaning "you are the copy."

**Worked example:**

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("before fork, pid=%d\n", getpid());   // pid=4821

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
    } else if (pid == 0) {
        printf("child: pid=%d, ppid=%d, fork returned %d\n",
               getpid(), getppid(), pid);          // pid=4822, ppid=4821, returned 0
    } else {
        printf("parent: pid=%d, fork returned %d\n",
               getpid(), pid);                      // pid=4821, returned 4822
    }
    return 0;
}
```

Two processes now exist, running the *same compiled binary*, from the *same point*, with separate copies of every variable from that point on. Order of the two prints is **not guaranteed** — the scheduler decides who runs first.

---

## 2. Fork patterns — the actual menu

These are the shapes `fork()` gets used in. Everything you'll see in real code is one of these five.

### A. fork + exec ("spawn a different program")
The shell's core loop. Child replaces itself with a new program; parent keeps running (usually after `wait`ing).
```c
pid_t pid = fork();
if (pid == 0) {
    execlp("ls", "ls", "-l", NULL);   // child becomes `ls`
    _exit(127);                        // only reached if execlp fails
} else {
    waitpid(pid, NULL, 0);            // parent blocks until ls finishes
}
```

### B. fork + wait, synchronous ("run this and block until it's done")
Same as A, structurally this is literally what `system()` does under the hood.

### C. fork only, no exec ("parallel copies of myself")
Used for preforking worker pools (old-school web servers): fork N children, each loops on `accept()` independently, no `exec` at all — they keep running the *same* binary.
```c
for (int i = 0; i < 4; i++) {
    if (fork() == 0) {
        worker_loop();   // child runs this and never returns
        _exit(0);
    }
}
// parent: 4 children now all running worker_loop() in parallel
```

### D. double fork ("daemonize — fully detach from the terminal")
```c
if (fork() > 0) exit(0);      // (1) original parent dies immediately
setsid();                      // (2) child becomes session leader, no controlling tty
if (fork() > 0) exit(0);      // (3) that child ALSO dies
// (4) only the grandchild survives — it's now an orphan, reparented to init (PID 1)
daemon_main();
```
Why the second fork? Step (2) makes the child a *session leader*, which means it could later acquire a controlling terminal. Killing it in step (3) guarantees the surviving grandchild can never become one — it's permanently detached. This is exactly how orphaning (§10) gets used deliberately instead of by accident.



> The goal is to make a process that has a new session but not the session leader. we the 1st child is not the leader but also not in a new session, when you make a new session you become the new leader so you the 2nd child doesnt work as well. the child however is not a leader but is in a different session(the 2nd childs one) so it works

 Kind of, after `setsid()`, the **first child becomes a session leader**:

```
Before setsid():

First child:
PID = 2000
PGID = 1000
SID  = 1000

(not a process group leader)
```

It is allowed to call:

```c
setsid();
```

because it is **not a process group leader**.

After `setsid()`:

```
First child:

PID  = 2000
PGID = 2000
SID  = 2000

(session leader)
```

So yes, you are right that:

> when you make a new session you become the new leader

Exactly.

---

Now the second fork:

```
First child (session leader)
PID = 2000
SID = 2000
      |
      fork()
      |
      +---- Parent
      |
      +---- Child
```

The second child gets a copy:

```
Second child:

PID  = 3000
PGID = 2000
SID  = 2000
```

Notice:

```
PID != SID
3000 != 2000
```

Therefore:

```
Second child is NOT a session leader
```

but it is still inside the new session:

```
Session 2000
|
+-- first child (dies)
|
+-- second child (daemon)
```

So your summary is basically correct:

> The first child is not a leader but is not in a new session. After `setsid()` it becomes a new session leader. The second child inherits that new session but is no longer the session leader.

One small wording fix:

> "the 2nd child's one"

The second child **does not create a new session**. It inherits the session created by the first child.

The flow is:

```
Original process
        |
        fork()
        |
        v
First child
(not process group leader)
        |
        setsid()
        |
        v
New session leader
        |
        fork()
        |
        v
Second child
(not session leader, but in the new session)
        |
        daemon
```

That is the entire trick. The second fork is not about making a new session; it is about throwing away the session leader role while keeping the new session.


### E. fork bomb (anti-pattern, know it to avoid it)
```c
while (1) fork();
```
Unbounded recursive forking exhausts the OS's process table / PID space. Mentioned only so you recognize it — never write this on purpose.

---

## 3. `exec()` family — replacing the image, not the process

```c
execl(path, arg0, arg1, ..., NULL);
execv(path, argv[]);
execle(path, arg0, ..., NULL, envp[]);
execve(path, argv[], envp[]);
execlp(file, arg0, ..., NULL);      // p = search $PATH
execvp(file, argv[]);
```

Critical fact: **`exec()` does not create a new process.** The PID stays identical. What changes:

- code segment, data segment, heap, stack — all wiped and replaced with the new program's
- program counter reset to the new program's entry point

What survives across `exec()`:
- the PID and PPID
- **open file descriptors** (unless marked `FD_CLOEXEC`) — this is why redirection (`cmd > file.txt`) works: the shell opens the file *before* calling `exec`, and the fd just carries over into the new program

Naming decoder:
| Letter | Meaning |
|---|---|
| `l` | args listed individually, `NULL`-terminated |
| `v` | args passed as a `char *argv[]` array |
| `p` | search `$PATH` for the executable |
| `e` | takes an explicit environment (`char *envp[]`), otherwise inherits caller's |

If `exec*()` succeeds, it **never returns** — there's no "old" process left to return to. If it returns at all, it failed, and you check `errno`.

---

## 4. `vfork()`

Like `fork()`, but the child does **not** get its own copy of the address space — it literally borrows the parent's, and the parent is suspended until the child calls `exec()` or `_exit()`. This made sense on old systems where copying page tables was expensive, before COW existed. It's fast, but dangerous: if the child modifies *any* variable, or returns from the function that called `vfork()`, before calling `exec`/`_exit`, behavior is undefined — you're corrupting the parent's live stack. Modern code should basically never use it; it survives mostly in exam questions and legacy code.

---

## 5. Process termination: `exit()` vs `_exit()`

```c
void exit(int status);     // libc function
void _exit(int status);    // raw syscall (also _Exit in <stdlib.h>)
```

`exit()` is layered on top of `_exit()`:

1. Runs any functions registered with `atexit()`, in reverse order of registration.
2. Flushes and closes `stdio` buffers (this is why a buffered `printf` without `\n` still shows up — `exit()` flushes it; `_exit()` would silently drop it).
3. Calls `_exit()` to actually hand control to the kernel.

`_exit()` skips all of that — it's the raw syscall: kernel closes the process's file descriptors, releases its memory, and stores the exit status. Nothing userspace-side runs.

**Why `_exit()` shows up right after `fork()` in the child:** if the child were to call `exit()` instead, it would flush `stdio` buffers it *inherited* from the parent — meaning any output the parent had already printf'd-but-not-yet-flushed could get printed a second time, once by each process. `_exit()` in the child avoids that double-flush bug.

Also: `return 42;` from `main()` is exactly equivalent to `exit(42);` — the C runtime wraps your `main` in something like `exit(main(argc, argv));`.

---

## 6. Exit status — what it actually *is*

This is the part worth being precise about, because "status" sounds abstract until you trace exactly where it lives.

**It is not a file. It is not heap or stack memory. It is a single small integer (0–255) stored inside the kernel's `task_struct` for that process.**

Walk through the full lifecycle with a concrete number:

1. Your program calls `exit(42)` (or returns 42 from `main`).
2. The kernel truncates that to 8 bits: `42 & 0xFF = 42`. (This is why exit codes are always 0–255 — anything you pass gets silently wrapped mod 256. `exit(300)` actually reports `44`.)
3. The kernel frees almost everything belonging to the process — its memory, its file descriptors, its address space — **except** the `task_struct` entry itself.
4. That leftover `task_struct` is kept around specifically to hold two things: the PID, and the packed status word (built from your exit code, or from a signal number if it was killed — see §12). The process is now called a **zombie** (§7).
5. The **parent** calls `wait()` or `waitpid()`, passing a pointer to an `int` in *its own* memory.
6. The kernel copies the packed status word out of the zombie's `task_struct` into that pointer.
7. Only now does the kernel destroy the last remnant — the `task_struct` — freeing the PID for reuse. This last step is called **reaping**.

So the answer to "where does it live": in kernel memory, attached to the dead-but-not-yet-reaped process's table entry, and it only reaches *your* process's memory (as a plain `int`) the moment `wait()`/`waitpid()` copies it over. Before that copy happens, your process cannot see it at all — there's no shared memory involved, it's a kernel→userspace copy across the syscall boundary, same as `read()` copying file bytes into your buffer.

**Why does the parent need this at all?** Because a child process is opaque from the outside — the parent can't peek at the child's memory or return value directly (separate address spaces). The exit status is the *one* narrow, deliberate channel the kernel provides for a child to report a small result (success/failure/error code) back to whoever's watching it. It's how `if child_process_failed:` gets implemented at the shell level — `$?` in bash is exactly this value.

---

## 7. Zombie process — the concrete picture

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();
    if (pid == 0) {
        _exit(7);              // child dies almost immediately
    } else {
        sleep(10);              // parent deliberately doesn't call wait() yet
        printf("done sleeping\n");
    }
    return 0;
}
```

During those 10 seconds, run `ps aux | grep defunct` in another terminal. You'll see something like:

```
oshan   4822  0.0  0.0      0     0 ?    Z    10:03   0:00 [a.out] <defunct>
```

`STAT = Z` — zombie. The child has already released its memory, its open files, everything — the `Z` state literally means "nothing left but the table entry and the exit status." It is not consuming CPU, and it is not "running" in any sense. It exists purely so the number `7` has somewhere to sit until the parent asks for it.

**Why this matters practically:** PIDs are a finite resource (check `cat /proc/sys/kernel/pid_max` — commonly 32768 or 4194304). A parent that forks children and never `wait()`s on them leaks a zombie table-entry per dead child. Enough of them, and you exhaust the PID space — new `fork()` calls start failing system-wide. This is the actual reason "always reap your children" is a rule, not just tidiness.

---

## 8. Orphan process

The mirror-image problem: what if the **parent** dies first, while the child is still running?

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();
    if (pid == 0) {
        sleep(5);
        printf("child's ppid is now %d\n", getppid());  // no longer the original parent
    } else {
        printf("parent %d exiting immediately\n", getpid());
        _exit(0);
    }
    return 0;
}
```

The moment the original parent exits, the still-running child is **reparented** — the kernel walks up and assigns it a new PPID, traditionally PID 1 (`init`/`systemd`), or on modern Linux, the nearest ancestor process marked as a "subreaper." That's the whole mechanism: an orphan isn't special-cased or paused, it just gets adopted so that *something* is guaranteed to eventually call `wait()` on it when it finishes — init reaps its adopted children automatically, which is exactly what stops orphans from becoming permanent zombies.

Note the two failure modes are opposites: a **zombie** is a dead child whose status nobody has collected yet; an **orphan** is a *live* child whose original parent is gone. A process can (briefly) be both — an orphan that then dies and waits for its new parent (init) to reap it.

---

## 9. `wait()` / `waitpid()` — and no, not really like a WaitGroup

```c
pid_t wait(int *status);                          // block for ANY child
pid_t waitpid(pid_t pid, int *status, int options); // block for a SPECIFIC child, or poll
```

`wait()` blocks the calling process until *some* child terminates, then returns that child's PID and (via the pointer) its packed status. `waitpid()` lets you target one specific PID, or pass `WNOHANG` to poll without blocking ("has it finished yet? — no? fine, don't wait, just tell me and I'll ask again later").

**On the Go comparison:** the surface goal is similar — "block until some concurrent unit(s) of work are done" — but the mechanism is fundamentally different, because of what's on each side of the boundary:

- A `sync.WaitGroup` coordinates **goroutines**, which live in the *same* process, *same* address space. The counter it decrements is just a shared integer in userspace memory; no kernel involvement is needed at all in the common case (it may use a futex to actually park the blocked goroutine, but there's no cross-process data transfer happening).
- `wait()` coordinates **processes**, which by definition have *separate* address spaces — the parent literally cannot see into the child's memory. So `wait()` has to be a genuine syscall: it's the kernel reaching into a different process's `task_struct` and copying a value across the process boundary into yours. It's less "counter reaches zero" and more "ask the kernel to hand me the death certificate of my child."

So: same *purpose* (synchronize on completion), completely different *category* of mechanism — userspace counter vs. kernel-mediated cross-process data transfer. `waitpid(pid, &status, WNOHANG)` in a loop is the closer analogy to polling a channel with a `select`/`default`, if you want a mental hook.

---

## 10. Decoding the status int — bit level

The `int status` that `wait()` fills in isn't the exit code directly — it's a packed word encoding *both* the exit code and *how* the process died. On Linux, the common encoding is:

```
 15                8 7        0
+--------------------+---------+
|     exit code      |  cause  |
+--------------------+---------+
```

- If the low byte (`status & 0x7F`) is `0` → the process exited normally via `exit()`/`return`, and the exit code is in the next byte up: `(status >> 8) & 0xFF`.
- If the low byte is nonzero (and not `0x7F`, which is reserved for "stopped") → the process was killed by a signal, and that low byte (`status & 0x7F`) *is* the signal number.

**Worked examples:**

| What happened | Resulting `status` (approx) | `status & 0x7F` | `(status >> 8) & 0xFF` |
|---|---|---|---|
| `exit(42)` | `0x2A00` = 10752 | `0` → normal exit | `42` |
| `exit(0)` | `0x0000` | `0` → normal exit | `0` |
| killed by `SIGKILL` (signal 9) | `9` | `9` → died by signal | n/a |
| killed by `SIGSEGV` (signal 11) | `11` | `11` → died by signal | n/a |

The macros just wrap that logic so you never hand-decode it:

```c
int status;
pid_t child = wait(&status);

if (WIFEXITED(status)) {
    printf("exited normally, code = %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("killed by signal %d\n", WTERMSIG(status));
}
```

`WIFEXITED` checks "was the low byte zero." `WEXITSTATUS` extracts the code byte. `WIFSIGNALED` / `WTERMSIG` do the signal-side equivalent. You should always use the macros, not the raw bit math above — the exact layout is technically implementation-defined per POSIX, even though the table above matches Linux/glibc in practice.

---

## 11. File descriptor inheritance — why pipes actually work

After `fork()`, the child gets a **copy of the fd table** — same fd numbers (0, 1, 2, 3...), each still pointing at the *same underlying open file description* in the kernel (which tracks things like the current read/write offset). Copy of the table, shared underlying object.

This is the entire mechanism behind pipes between parent and child:

```c
int fd[2];
pipe(fd);              // fd[0] = read end, fd[1] = write end

pid_t pid = fork();
if (pid == 0) {
    close(fd[1]);                  // child doesn't write
    char buf[64];
    read(fd[0], buf, sizeof(buf)); // reads whatever parent wrote
} else {
    close(fd[0]);                  // parent doesn't read
    write(fd[1], "hi", 2);
    wait(NULL);
}
```

Before `fork()`, there's one process with two fds pointing at one pipe (an in-kernel buffer). After `fork()`, there are *two* processes, each with a copy of those two fd numbers — but both copies still point at the *same* pipe buffer. That shared underlying object is what lets a `write()` in one process show up in a `read()` in the other. Nothing about the pipe itself is duplicated — only the table of "which fd number means which open file" is duplicated. This is also exactly why you `close()` the unused end in each process: leaving the write end open in the reading process means the kernel never sees "all writers closed," so a `read()` waiting for EOF blocks forever.

---

## Cheat sheet

| Call | Creates new process? | Returns |
|---|---|---|
| `fork()` | yes | twice: `0` in child, child PID in parent |
| `exec*()` | no — replaces image | never (on success) |
| `vfork()` | yes, but shares memory until exec/_exit | same as fork, but risky |
| `exit(n)` | — | doesn't return; runs atexit + flushes stdio |
| `_exit(n)` | — | doesn't return; raw syscall, no cleanup |
| `wait(&status)` | — | PID of a terminated child, blocking |
| `waitpid(pid, &status, opts)` | — | PID of *that* child, or `0` with `WNOHANG` if not done |
