---
date: 2026-07-03
---


### Trying to understand concurrency
>[What every programmer should know about memeory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) [coding jesus video about the paper](https://youtu.be/SQpS3NuOFT8?si=no1jCOZCYo4mzhXP), 

## Why concurrency had to happen

Go back to the framing in Section 3's intro, which you already covered: CPU core frequency kept climbing through the 90s while memory speed didn't keep pace. 
https://youtu.be/3RvkfuXUv1c?si=Ak62SbF_jXQ6yn_Q
[good](https://youtu.be/hwTYDQ0zZOw?si=vh4_ZSU_MHiLPRXC)
---
### Invention of threads

Threads are best understood as a solution to concurrency, 
:
For most of computing history (1960s–early 2000s), there were only single core CPUs
What multicore CPU does is genuine hardware parallelism and it didn't become mainstream until roughly 2005, when Intel and AMD shipped the first mainstream dualcore chips. 


##### A rough timeline
- **1960s**: Multiprogramming (OS/360, THE) and time-sharing (CTSS, Multics) establish concurrency as a core OS idea, purely to keep a single CPU busy and let multiple users share it.
- **1970s**: Xerox PARC's Cedar/Mesa environment builds "lightweight processes" for responsive, structured interactive software — an important direct ancestor of the modern thread.
- **Mid-1980s**: CMU's Mach microkernel formalizes the split between a "task" (an address space/resource container) and a "thread" (a unit of execution within it) — the conceptual model most modern OSes still use.
- **Early-mid 1990s**: Native thread support goes mainstream — Windows NT (1993) ships with kernel threads built in; POSIX threads (pthreads) are standardized in 1995 for Unix-like systems.
- **~2005**: Intel and AMD ship the first mainstream dual-core desktop/server CPUs. Only *now* does running threads on genuinely separate execution units become a mass-market reality.

 So threads had roughly **three to four decades of useful life** solving I/O-latency-hiding, responsiveness, and program-structuring problems before multicore hardware gave them a second job: actual parallel speedup. Multicore didn't invent the need for threads — it just added a new, very compelling reason to keep using something that already existed.



So chip vendors had two levers left: make one core smarter (diminishing returns — pipelines were already deep, out-of-order execution was already aggressive), or put more cores on the die. They chose "more cores." That's the entire reason concurrency became a programmer's problem instead of a hardware team's problem. The transistors kept coming (Moore's Law didn't stop), but they stopped translating into single-thread speed and started translating into *core count*. If your program is single-threaded, it stops getting faster for free. That's "why concurrency" in one sentence: **the hardware stopped giving single-threaded programs a free ride, so software has to explicitly use the parallelism that's sitting there.**

## The four levels of "more than one thing running at once"

The paper actually gives you a clean hierarchy in Figure 3.3 and section 3.5.3. This matters because each level shares a *different amount* of hardware, and that determines what kind of concurrency bugs and slowdowns you get:

1. **Hyper-threads (SMT)** — share everything on a core except the register set. Same L1, same execution units, same everything. Two threads here are basically time-slicing the same physical resources.
2. **Cores on the same die** — separate L1 (and sometimes separate L2), but typically share a higher-level cache (L2 or L3) and always share the path to memory (FSB or memory controller).
3. **Separate sockets (SMP)** — separate caches entirely, but historically shared one Northbridge/FSB to memory (Figure 2.1).
4. **NUMA nodes** — separate memory controllers entirely, each with locally-attached RAM, connected via an interconnect (HyperTransport/CSI).

Notice the pattern: as you go from 1→4, you share *less* hardware, but accessing "the other guy's" resource gets progressively *more expensive*. Hyper-thread siblings share cache for free (no extra latency) but fight for it. NUMA nodes never fight over the same DRAM, but reaching across nodes costs a real, measurable NUMA factor (the paper measured 20-30% slower remote reads in section 5.4).

This is the single most important mental model for concurrent programming on real hardware: **concurrency isn't binary, it's a spectrum of "how much do these two flows of execution share," and every level of sharing has a cost.**

## The core problem: shared memory needs coherency

This is where your Chapter 3 work pays off directly. The moment you have more than one core that can touch the same memory address, you need MESI (or something like it) to guarantee that all processors agree on the current value of memory at all times. You already know the four states — Modified, Exclusive, Shared, Invalid — and the transitions in Figure 3.18.

Here's the concurrency-specific reframe: **every write to shared data that another core has cached triggers an RFO (Request For Ownership) message.** RFOs are not free — they mean stopping, broadcasting on the bus, waiting for every other cache to invalidate, and only then proceeding. The paper is blunt about this: "filling caches is still expensive but now we also have to look out for RFO messages. Whenever such a message has to be sent things are going to be slow."

This is the physical reason race conditions exist at the hardware level, not just the software level. Picture two cores, both with a variable `x` cached in Shared state, both doing `x++`:

- Core A reads x=5 from its cache (fine, it's Shared, no bus traffic needed for a read)
- Core B reads x=5 from its cache too (same)
- Core A computes 6, has to send an RFO to invalidate B's copy, writes 6
- Core B computes 6 (using its now-stale read of 5), sends its own RFO, writes 6

Final result: 6, even though two increments happened. One update is lost. This isn't a compiler bug or a language bug — it's a direct consequence of caches existing and being read before the "true" hardware serialization point (the RFO) has happened. **This is why "concurrent" and "correct" are not the same word**, and why you need explicit synchronization primitives.

## Atomic operations: the hardware's answer to races

Section 6.4.2 covers this directly. Processors expose special instructions that guarantee the read-modify-write sequence happens as one indivisible unit with respect to other processors. Two hardware philosophies exist:

**Load-Locked/Store-Conditional (LL/SC)** — used by most RISC architectures. `LL` starts a transaction by loading a value and marking the cache line as being watched. `SC` writes a new value back but *only succeeds if nobody touched that cache line in between*. If it fails, you loop and retry.

**Compare-And-Swap (CAS)** — used by x86/x86-64. You give it: an address, an expected old value, a new value. It writes the new value only if the current value still matches what you expected; otherwise it tells you it failed and you retry with a fresh read.

Both approaches are equivalent in power, and virtually all higher-level atomic operations (`__sync_fetch_and_add`, atomic increments, etc.) can be built from either. But here's a subtlety the paper measured directly (Figure 6.12 and the table right after it) that's a great concrete lesson: on x86, using a CAS loop to implement something as simple as an atomic increment is **~3x slower** than using the native atomic add instruction:

- Exchange-and-add: 0.23s
- Add-and-fetch: 0.21s
- CAS loop: 0.73s

Why? A CAS loop does two separate memory operations (load, then a conditional compare-and-store) and, under contention, has to *retry* — meaning it can burn through multiple failed RFO round-trips before it succeeds. A native atomic add is one bus transaction, done. The lesson: **CAS is the universal building block, but it is not free, and reaching for it by default when a narrower atomic op exists is a real performance mistake.**

There's also a neat trick the paper describes for avoiding atomic-instruction cost entirely in the *uncontended* case: on x86, you conditionally jump over the `lock` prefix if you know only one thread is active. An atomic op costs ~200 cycles; a mispredicted branch is much cheaper than that, so this pays off whenever single-threaded execution is common.

### The ABA problem — why lock-free is harder than it looks

This is one of the best "gotcha" lessons in the whole paper (section 8.1). Say you build a lock-free stack (LIFO) using CAS: pop reads the top pointer, and if nothing else changed it, swaps in the next element. Sounds airtight. But consider this interleaving:

1. Thread 1 starts `pop()`, reads `top = A` (A's next pointer is B).
2. Thread 1 gets preempted right before its CAS.
3. Thread 2 runs: pops A, pushes some new node, then happens to push A back onto the stack. Now `top = A` again — but A's internal `next` pointer has been rewritten to point somewhere completely different.
4. Thread 1 resumes. Its CAS checks "is top still A?" — yes! So it succeeds... but it installs A's *stale* next pointer as the new top, corrupting the stack.

The address came back around to the same value (A→X→A), so the CAS can't tell anything happened in between. That's the "ABA problem." It's exactly why naive CAS-based lock-free structures are dangerous, and why real implementations need extra machinery — generation counters, double-word CAS, or hazard pointers/safe memory reclamation. This is precisely the motivation the paper gives for wanting **transactional memory** as a hardware primitive (more on that below) — it sidesteps ABA entirely because it tracks the whole read-set of a transaction, not just one address's current value.

## False sharing: the concurrency bug that isn't a race

This is arguably the single most practically important lesson for you as someone who just did a deep dive on cache lines. **Two threads can write to two completely independent variables and still destroy each other's performance**, purely because those variables happen to live on the same 64-byte cache line.

The paper's benchmark (section 6.4.1, Figure 6.10) makes this brutally concrete: four threads, each just incrementing its own private counter 500 million times, no logical sharing at all.

- Counters on separate cache lines: scales basically linearly, near-zero overhead.
- Counters packed onto the same cache line: **390% slower with 2 threads, 1147% slower with 4 threads.**

Why? Every increment requires the cache line to be in Modified/Exclusive state locally. The moment Thread B writes to "its" part of the line, it has to RFO the line away from Thread A — even though A and B never touch each other's data. The line just bounces between caches in ping-pong fashion, and every bounce costs a full coherency round-trip. This is called **false sharing**, and it's invisible in your source code — there's no shared variable, no lock, no race condition to reason about. It only exists at the cache-line granularity.

The fixes, straight from the paper:
- Separate hot, frequently-written-by-different-threads variables onto their own cache lines (padding structs to 64 bytes, `__attribute__((aligned(64)))`).
- If a variable is genuinely only ever touched by one thread, make it **thread-local** (`__thread` in gcc / TLS) instead of a shared global — removes it from the coherency picture entirely.
- Keep read-only/const data grouped separately from read-write data, since read-only data can happily sit in Shared state in every cache with zero RFO traffic.

## Bandwidth: the resource nobody remembers is finite

Even once you've eliminated races and false sharing, there's a hard physical ceiling: **the bus to memory has a fixed bandwidth, and every core sharing that bus divides it up.** Section 6.4.3 and the multi-thread bandwidth graphs (3.19–3.22) show this isn't hypothetical — it's measured. With a working set too large for the caches:

- 2 threads reading sequentially: only 1.69x speedup instead of the theoretical 2x
- 4 threads: only 2.98x for reads, and for a random-access/write-heavy pattern, only **1.65x with 4 threads** — meaning going from 2 threads to 4 barely helped at all.

This is why "just add more threads" stops working past a certain point for memory-bound code: you're not compute-limited, you're bus-limited, and more cores just means more contenders for the same pipe. This is also the direct motivation for NUMA — give each processor its *own* memory controller and local RAM so bandwidth scales with core count instead of being a shared bottleneck (Figure 2.3).

## From hardware reality to actual thread management

Once you accept that placement matters this much, "concurrent programming" stops being purely about locks and starts being about *where* the OS puts your threads relative to the data and each other. The paper gives two opposite scheduling scenarios (Figures 6.13/6.14) that are worth internalizing as a pair:

- **Threads sharing the same data, scheduled on different cores/caches** → bad. Each core has to independently fetch the same data into its own cache from memory; effectively doubles memory traffic that didn't need to happen.
- **Threads working on totally independent data, scheduled on cores that share a cache** → also bad. They compete for the same limited cache capacity and evict each other's working set, plus they compete for the same slice of bus bandwidth.

The tool for controlling this is **thread affinity** — `sched_setaffinity`/`pthread_setaffinity_np` and `cpu_set_t`. The default OS scheduler has zero insight into your data-sharing patterns; it's just balancing load. If you know two threads work on the same dataset, you can pin them to sibling hyper-threads or cores sharing an L2/L3 so their memory traffic collapses into shared cache hits instead of duplicated fetches. If they're independent, you pin them apart.

## NUMA-aware threading

Extend this one more level: on a NUMA machine, it's not just "which cache" but "which memory node." The default Linux policy actually *stripes* memory allocation across nodes to avoid starving any one node — which is safe but not optimal. Section 6.5 gives you the tools to do better:

- `set_mempolicy`/`mbind` to control which node new allocations land on (`MPOL_BIND`, `MPOL_PREFERRED`, `MPOL_INTERLEAVE`).
- The general recipe: figure out which node a thread is running on (`sched_getcpu` → `NUMA_cpu_to_memnode`), then allocate that thread's working data local to that node before spinning it up.
- Critical ordering point: set a thread's CPU affinity *before* it allocates its memory, so first-touch allocation lands locally instead of striping across nodes by default.

The "why" here is the same NUMA factor idea from Chapter 2, just now applied deliberately: a thread computing on remote-node memory pays the interconnect tax on every cache miss that goes all the way to DRAM, stacked on top of everything else.

## Where this is heading: beyond locks and CAS

The paper closes (section 8) by pointing at where hardware concurrency support was headed next, which is a nice way to frame the arc of this whole lesson. Locks (mutual exclusion) work but serialize everything and can deadlock. CAS-based lock-free structures avoid locks but run into ABA and make anything beyond a single-word update awkward (real double-word CAS is rare and limited). The proposed next step is **hardware transactional memory**: mark a block of loads/stores as one transaction; the processor tracks the whole read/write set via an extra transactional cache layered on the coherency protocol, and the transaction either commits atomically or aborts and retries if any of its addresses were touched by someone else in the meantime. It's essentially generalizing the MESI machinery you already know into "watch a whole set of cache lines, not just one," which cleanly solves ABA and multi-location atomicity in one primitive.

## The mental model to keep

If you strip away all the API details, concurrent programming built up from this paper is really one idea repeated at every layer: **sharing is never free, and the cost is proportional to how much hardware two flows of execution actually share and how often they contend for it.** Threads sharing a core fight over execution units. Threads sharing a cache line fight over coherency state. Threads sharing a bus fight over bandwidth. Threads sharing a NUMA node's neighbor fight over interconnect latency. Every synchronization primitive — atomics, locks, thread-local storage, affinity, memory policy, and eventually transactional memory — exists to either avoid unnecessary contention or make necessary contention as cheap and correct as possible.

Where do you want to go next — do you want to work through an actual pthreads example applying this stuff (affinity + avoiding false sharing), or keep going conceptually into Chapter 4 (virtual memory/TLB), which is next in your sequence?
