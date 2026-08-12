# Threads, Demystified: Why Processes Alone Aren't Enough

## Concurrency: A Quick Recap

If a system has one CPU and several running processes, the operating system makes those processes take turns on the CPU — switching between them so quickly that it looks, to the user, like everything is running at the same time. This trick is called **concurrency**, and together with CPU scheduling, it's what gives us the smooth multitasking experience we take for granted.

Concurrency has a second, less obvious job beyond making multitasking *feel* seamless: maximizing how much of the CPU actually gets used. A process being "in execution" doesn't mean it's constantly using the CPU — even while the CPU is allocated to it. A process might be sitting there waiting on an I/O resource, unable to execute a single instruction until that resource is ready. Leaving the CPU allocated to a process in that state would be wasteful. It makes far more sense to hand the CPU to a different process — one that's actually ready to run — and that's exactly what concurrency lets the operating system do: fill in the gaps left by processes that can't currently use the CPU.

## The Limits of Treating the Process as the Basic Unit

So far, processes have been treated as the most basic unit of execution — there's nothing "smaller" for the scheduler to work with. That has a real consequence: two tasks *within* the same process — say, two functions in the same program — can't run concurrently under the traditional process model.

The reason traces back to something fundamental about processes: each one has exactly one program counter, which records where execution was interrupted so it can resume later. With only one program counter per process, there's no way to track two separate interruption points for two different pieces of code running inside that same process. One of them has to simply wait for the other to finish.

A natural question follows: if we want two pieces of code to cooperate and run concurrently, why not just split them into separate processes and let them coordinate through interprocess communication? That's a valid option, but IPC isn't always the natural fit — sometimes different pieces of code contributing to the same goal are so tightly related that splitting them into separate processes just feels like the wrong abstraction.

## A Motivating Example: The Bottlenecked Server

Consider a server listening on port 3000 whose only job is to return a client's profile image. For each request, the server accepts the connection, searches for and reads the corresponding image off disk, and once it's loaded into memory, sends it back.

Reading a file from disk is a comparatively expensive I/O operation, so processing a request takes noticeably longer than simply accepting it or sending the response. For one client at a time, that's a non-issue — the whole thing takes a few milliseconds. But run the same steps sequentially against many clients, and problems appear fast.

If five requests land at roughly the same time, the server accepts the first one quickly, then spends real time processing it — during which the second request just sits there waiting, and the third waits behind that, and so on. Five requests is a minor inconvenience. A thousand requests exposes the real issue: long stretches where the CPU sits completely idle, doing nothing, while later requests wait for their turn — wasted computational time that could have gone toward accepting or beginning to process other requests. This is called a **blocking effect**: one task can't execute because another is running, even though the two have nothing to do with each other.

### The Old Fix: One Process Per Client

The traditional answer to this was a **listener process** that accepts incoming requests but doesn't handle them directly. Instead, for every client, it spawns an entirely new process to serve that specific request. Because the listener keeps running concurrently with whatever process is busy serving a previous client, new requests don't have to wait behind old ones.

Clever as it is, this approach has real costs. Each process is a self-contained context with its own address space, and creating a full process per client scales poorly — fine for a few hundred clients, expensive in memory for thousands. Spawning a process also isn't free computationally: it eats into the very idle CPU time it's meant to reclaim. And if the server needs to track any shared global state, every one of those child processes needs to be synchronized somehow, which drags IPC back into the picture and complicates the whole implementation.

## Enter the Thread

Processes are too useful a concept to throw away, but they can be adapted to allow concurrency *inside* a single process. Recall that a process bundles together an ID, a program counter, a register set, an address space, and other resources like open files. The single program counter was the bottleneck — it's the reason the OS's Process Control Block can only track one point of execution per process.

The fix: stop tying the program counter to the process itself, and instead give a program counter to each inner executable entity that needs to run concurrently within that process. Those inner entities are **threads**. Once a process is no longer limited to a single program counter, a new thread can be created any time a piece of code within that process needs to run concurrently with another — solving the blocking problem without the overhead of spawning a whole new process.

### What Threads Share, and What They Don't

Threads within the same process share that process's entire address space. What they *can't* share is CPU state, since threads get interrupted and resumed just like processes do, and the OS needs to save and restore each one's state independently. That means each thread needs its own program counter, its own register set, flags, accumulators — and, importantly, its own stack pointer.

The stack matters here specifically because it's how local variables get organized in memory. If two functions running concurrently shared a single stack, they'd constantly overwrite each other's data — so each thread gets its own stack.

Having a separate stack doesn't mean threads are walled off from each other, though. The address space is a property of the *process*, not of any individual thread, and since all the stacks live inside that same shared address space, nothing technically stops one thread from reading or writing another thread's stack. Whether it's a good idea is a different question entirely — generally, unless you really know what you're doing, it's best avoided. If threads genuinely need to share data, the **heap** is the more natural place for that, since heap memory isn't organized in a predictable, thread-specific way to begin with. Even there, care is still required: one thread reading from a block of memory while another is writing to it is a recipe for disaster, which is exactly why synchronization mechanisms for concurrent access are baked directly into hardware.

Beyond the isolation of address spaces themselves — processes still don't share memory with each other by default — this means that if threads *belonging to different processes* need to communicate, IPC is still the tool for the job. Threads solve concurrency within a process; they don't replace IPC between processes.

## Representing Threads: Back to the PCB

Introducing threads means the Process Control Block needs rethinking. One option would be a whole separate structure just for threads, but the more common approach — and the one used by the Linux kernel — is to represent both processes and threads with the exact same structure, called a **task**. This is a deliberately elegant piece of naming and design: it erases the distinction between processes and threads at the data-structure level, so instead of describing concurrency as "alternating between processes, threads, or both," you can just describe it as alternating between tasks. That also means a single scheduler can handle both processes and threads without any special-casing.

This design requires every process to have at least one thread, called the **main thread**, which the OS creates automatically the moment a process is created. That way, even a process that never spins up any additional threads can still be scheduled — the OS just schedules its main thread.

Whether a process starts with a fixed set of extra threads or creates them dynamically depends on the operating system's implementation. Most mainstream systems favor the dynamic approach: a process starts single-threaded, and additional threads get created at runtime as needed. This adds complexity to the OS, since it has to support system calls for dynamic thread creation, but it's the practical choice — in most real programs, it's simply impossible to know at compile time how many threads will actually be needed.

One more wrinkle worth knowing: in many implementations, the main thread is what identifies the process itself. If other threads have been spawned and the main thread finishes executing, every other thread in that process terminates immediately along with it. There are ways to work around this, but that's a topic for another time.

### Back to the Server

Applying threads to the earlier server example: instead of spawning a whole new process per client, the server can spawn a new *thread* per client. This gets a similar effect to the multi-process approach — new requests no longer wait behind old ones — but at a fraction of the memory cost. There's often a performance win too, since the system calls used to create threads are generally faster than the ones used to create entire new processes, for reasons that should now be fairly intuitive.

## What a Thread Is (and Isn't)

It helps to be precise here: **a thread is not a function.** Any code a thread executes lives in the text section of the process's memory layout. A thread doesn't *contain* code — it *points to* code, via its program counter. That means multiple threads can point to the exact same executable code at the same time. This is exactly what happens in the server example: every request-handling thread doesn't hold its own private copy of a function, it points to the same function sitting in memory.

Is it dangerous for multiple threads to access that shared code region simultaneously? No — executable code and constants live in the text and data sections, which are never written to at runtime. Anything specific to a particular thread and generated while the program runs lives on that thread's stack or on the heap instead. Reading shared, never-modified executable code concurrently is completely safe.

This also explains a detail that trips people up in lower-level languages like C: spawning a thread involves passing a *pointer to a function*. That's not some kind of asynchronous function call — it's simply passing the memory address where the thread's code begins.

There are two useful ways to define a thread, depending on the angle you're looking from:

- From the **operating system's** perspective, a thread is essentially what a process was in earlier discussions — the most basic unit of execution.
- From a **developer's** perspective, a thread is a mechanism for telling the operating system that certain pieces of code inside a program can run concurrently.

A third framing worth keeping in mind: threads are sometimes described as **lightweight processes** — easier and faster to create than a full process, while still giving you concurrency.

Everything covered here holds even on a system with a single CPU core. The next natural question — what threads gain you specifically on systems with *multiple* cores, and how that connects to true parallelism — is a topic of its own.