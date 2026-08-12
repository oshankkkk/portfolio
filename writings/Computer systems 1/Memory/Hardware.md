---
Title: Understanding memory hardware
date: 2026-07-03
---


 ==My questions ==
Chips and chipsets
North bridge and south bridge
Buses and data and signals(they are the same thing ryt like signal is a data ye)
64 bits system CPU can support more RAM compared to 32??
how the memory controller works in the north bridge


# Chapter 2: Commodity Hardware Today  Explained

Let me walk you through this chapter like I'm teaching it to you. The core question Chapter 2 answers is: **"How does RAM actually get connected to the CPU, and why does that create bottlenecks?"**

## 2.0 The Big Picture First

Before any technical details, understand the motivation: computers used to be simple and balanced — CPU, memory, and storage all improved at similar rates. Then CPUs got fast *much* quicker than memory did. This created a "bottleneck" — the CPU sits around waiting for data. Chapter 2 explains the physical wiring/architecture that causes this bottleneck.

## 2.1 The Northbridge/Southbridge Architecture (Classic PC Design)

This is the foundational concept. A traditional motherboard chipset has **two chips**:

**Northbridge**
- Sits between the CPU(s) and RAM
- Contains the **memory controller** (this determines what RAM type — DDR, SDRAM, etc. — the system can use)
- All CPUs connect to the Northbridge via the **Front Side Bus (FSB)** — a shared bus
- Also connects onward to the Southbridge

**Southbridge**
- Handles all the I/O device traffic: PCI, PCI Express, SATA, USB, and older stuff like PATA/serial/parallel ports
- Everything the Southbridge talks to has to route *through* the Northbridge to reach the CPU or RAM

### Why this design creates bottlenecks — 4 key consequences:

1. **CPU-to-CPU communication** goes over the same bus used to talk to the Northbridge (shared resource)
2. **All RAM traffic** must pass through the Northbridge (single choke point)
3. **RAM has only one port** — can't be accessed from two places simultaneously (unlike specialized network-router RAM, which is multi-port but expensive)
4. **Southbridge devices talking to CPU** get routed through Northbridge too — more contention

**Key insight:** everything funnels through the Northbridge. That's the bottleneck.

## 2.1 DMA — Direct Memory Access

Originally, if a device (disk, network card) needed to move data to/from RAM, the *CPU* had to shuttle every byte — hugely wasteful. **DMA** lets devices talk to RAM directly through the Northbridge, without bothering the CPU.

- **Good:** frees up the CPU
- **Bad:** DMA transfers now *compete* with the CPU for Northbridge/RAM bandwidth. So even though DMA is "free" for the CPU's instruction execution, it still eats into the shared memory bandwidth budget.

## 2.2 The Second Bottleneck: Northbridge-to-RAM Bus

Even after data reaches the Northbridge, it has to go to the actual RAM chips over another bus.

- **Older systems:** one bus to all RAM chips → no parallelism, everything serialized
- **Modern systems (DDR2 era):** two separate **channels** double the bandwidth, and the Northbridge *interleaves* access across them
- **Even newer (FB-DRAM):** more channels

**Access patterns matter** because of this — how you traverse memory (sequential vs. random) interacts with how many channels/banks can work in parallel.

## 2.3 Alternative Architecture: External Memory Controllers

On pricier systems, instead of the memory controller living *inside* the Northbridge, the Northbridge connects out to **multiple separate external memory controllers**, each with its own RAM.

- **Advantage:** multiple independent memory buses = more total bandwidth, more supportable RAM
- **Bottleneck shifts** to the *internal bandwidth of the Northbridge itself* — how fast can it shuffle data between all these external controllers and the CPUs

This is a stepping stone to the next (and most important) architecture.

## 2.4 Integrated Memory Controllers → NUMA (The Big One)

This is the most conceptually important part of the chapter.

**The idea:** instead of one central memory controller (in the Northbridge or external), put a memory controller **inside each CPU**, and give each CPU its own local RAM.

- Pioneered commercially by **AMD Opteron**
- Intel followed with **CSI** (Common System Interface) starting with Nehalem

**Advantages:**
- On a quad-CPU machine, memory bandwidth is *quadrupled* — no single Northbridge choke point
- No need for one enormous, expensive, high-bandwidth Northbridge

**The catch — this is where NUMA comes from:**

Since each CPU has its own *local* memory, but all CPUs still need to be able to access *all* memory in the system (a single unified address space), a CPU sometimes needs to read memory that's physically attached to a *different* CPU. This access has to travel across an **interconnect** between processors (AMD used HyperTransport).

This makes memory access **Non-Uniform** — hence **NUMA: Non-Uniform Memory Architecture**.

- **Local memory access** = full speed
- **Remote memory access** = must cross one or more interconnects, each hop adds cost

### NUMA Factor
This is the key term: the **NUMA factor** describes *how much slower* remote memory access is compared to local. 

- Example from the chapter: a 4-CPU layout might have "two levels" — CPUs directly adjacent (1 hop) vs. two interconnects away (2 hops)
- More complex machines (IBM x445, SGI Altix) can have many more levels/hops, with progressively worse NUMA factors
- Big/expensive server systems can have very high NUMA factors depending on topology and load

**Important prediction the chapter makes (written 2007):** "It is expected that, from late 2008 on, every SMP machine will use NUMA." This tells you NUMA wasn't a niche thing — it became the *default* architecture for any multi-socket machine.

## 2.5 Why Programmers Need to Care (the thread that ties Ch.2 together)

The whole chapter is setting up this punchline: **hardware topology is no longer invisible to software.** In the old unified-bus world, "memory access" had one cost. Now:

- Which CPU your thread runs on matters
- Which memory bank your data lives in matters
- The OS and the *programmer* need to be aware of locality (this foreshadows Chapter 5's NUMA support discussion and Chapter 6's optimization advice)

## Quick Mental Model to Remember This Chapter

```
OLD WORLD:
CPU ↔ Northbridge(Memory Controller) ↔ RAM
              ↓
         Southbridge ↔ [USB, SATA, PCI-E, etc.]

PROBLEM: Northbridge = single choke point for everything

NEW WORLD (NUMA):
CPU1(+MC)↔RAM1 ←interconnect→ CPU2(+MC)↔RAM2
   ↕ still needs Southbridge for I/O somewhere

BENEFIT: parallel bandwidth
COST: non-uniform access — "distance" to memory now matters
```

## Summary Table

| Design               | Memory Controller Location                       | Bandwidth Scaling         | Weakness                                               |
| -------------------- | ------------------------------------------------ | ------------------------- | ------------------------------------------------------ |
| Classic Northbridge  | Inside Northbridge                               | Poor — single bus         | Northbridge bottleneck                                 |
| External controllers | Multiple, outside Northbridge, still centralized | Better                    | Northbridge internal bandwidth still limits            |
| Integrated (NUMA)    | Inside each CPU                                  | Best — scales with # CPUs | Non-uniform costs; remote access penalty (NUMA factor) |

That's the whole architectural arc of Chapter 2: **centralized bus → shared bottleneck → distributed memory controllers → NUMA and its associated "distance costs."** Everything Drepper discusses later (cache coherency traffic between nodes, NUMA-aware allocation, thread/memory affinity) is a direct consequence of the hardware picture built here.

Want me to go deeper on any specific piece — like the NUMA factor math, or how this connects to what you'll see in Chapter 5's OS-level NUMA support?

# Numa effect in programmers

## What "invisible to software" means

In the **old, unified-memory-bus world**, when your program did something like:

```c
int x = array[1000000];
```

...it didn't matter *which* physical RAM chip `array[1000000]` lived in. Every byte of RAM was reachable through the same path (CPU → Northbridge → RAM), at the same cost, every time. As a programmer, you literally could not make a "wrong" choice about where your data sat in memory — there was only one kind of memory access, so there was nothing to optimize for at that level. The hardware's internal wiring was irrelevant to you; you could pretend memory was just "one big flat array with one speed."

## What changes with NUMA

Now each CPU has **its own local RAM**, and reaching another CPU's RAM means hopping across an interconnect. So the *same line of code* can behave differently depending on circumstances outside the code itself:

```c
int x = array[1000000];
```

- If `array` happens to be allocated in memory local to the CPU your thread is running on → **fast**
- If `array` was allocated by a different thread (running on a different CPU) and now *your* thread reads it → **slow**, because you're paying the NUMA-factor penalty crossing the interconnect
- If the OS scheduler moves your thread to a different CPU core mid-run → the memory that used to be "local" is now "remote," and you didn't do anything wrong in your code — the hardware topology just shifted under you

## Concretely, what does "no longer invisible" require you to do?

It means the programmer/OS now has to actively think about things like:

1. **Where does this thread run?** (which CPU/core)
2. **Where was this memory allocated?** (which RAM bank, attached to which CPU)
3. **Are those two things close to each other or far apart?**

None of these questions existed — or mattered — in the old Northbridge-only world. Now they directly affect performance, sometimes by a large factor. That's what later chapters (5 and 6) are about: giving programmers tools (`sched_setaffinity`, memory policies like `MPOL_BIND`, checking `/sys/devices/system/node/`) to explicitly control *where* threads run and *where* their memory lives — because leaving it to chance can silently make your program much slower.

**One-line summary:** Before NUMA, "memory" was one thing. After NUMA, "memory" became "memory *relative to where you're standing*," and that relativity is something your code now has to account for instead of ignoring.


# RAM Diffs
Yes — Section 2.1 and 2.2 cover RAM types (SRAM vs DRAM) and how DRAM actually gets accessed. Strip away the transistor-level physics and there are several genuinely important CS concepts hiding in there. Let me pull them out.

## 1. The SRAM vs. DRAM Tradeoff — Why Caches Exist At All

This is honestly the most important idea in the whole section, and it's the reason the rest of the paper exists.

- **SRAM** = fast, but expensive and low-density (needs more transistors per bit)
- **DRAM** = slow, but cheap and high-density (way more bits per chip, for the money)

**The CS concept:** you can't have "all fast memory" because of cost/density economics. This is a classic **engineering tradeoff space** — and it's *the* justification for the entire memory hierarchy (registers → L1 → L2 → L3 → RAM → disk). Every level exists because no single technology gives you both speed and capacity affordably. Once you internalize this tradeoff, the whole reason for caching clicks into place: caches are small pools of expensive-but-fast SRAM sitting in front of huge pools of cheap-but-slow DRAM.

## 2. DRAM Has "Invisible" Background Overhead — Refresh

DRAM cells leak charge and must be **refreshed** constantly (every 64ms) or they lose their data. While a row is being refreshed, it can't be accessed.

**The CS concept:** this is a good example of **overhead that's invisible in your source code but still eats into your performance budget**. You never write a line of code that says "refresh memory" — the memory controller does it silently in the background — yet it can stall a meaningful percentage of memory accesses under heavy workloads. This is a recurring theme in systems programming: costs that don't show up anywhere in your program's logic but absolutely show up in your program's runtime.

## 3. Row/Column Addressing → This Is Where "Locality" Comes From At the Hardware Level

DRAM chips organize their storage cells into a grid (rows and columns). To read *anything*, the memory controller must:
1. **Activate/open a row** (this has a cost — like "loading a page into a buffer")
2. **Select a column** within that already-open row (this is cheap once the row is open)

**The CS concept — this is the important one:** as long as you keep reading from the *same open row*, subsequent accesses are cheap. The moment you jump to a different row, you pay the "open a new row" cost again (precharge + activate).

This is literally the hardware-level reason why **sequential memory access is fast and random memory access is slow** — a fact you already know intuitively from caching, but here's where it originates *even below the cache*, at the DRAM chip level. It's the same "locality of reference" principle you see everywhere in CS (page tables, caches, disk I/O), just showing up one layer deeper than you'd expect.

This directly foreshadows the massive performance graphs later in the paper (Chapter 3) comparing sequential vs. random access — now you know *why* that gap exists at multiple levels of the stack simultaneously (cache line locality AND DRAM row locality both reward sequential access).

## 4. Decoupling Internal Speed from External (Bus) Speed — The DDR/DDR2/DDR3 Trick

The paper explains SDR → DDR → DDR2 → DDR3 evolution. Skip the marketing-number nonsense entirely. The one real concept:

**Each generation doesn't actually make the memory cells themselves faster.** Instead, they add a **buffer** that collects multiple bits from the (still-slow) cell array and shoves them out the door faster on the external bus, by transferring more bits per clock edge.

**The CS concept:** this is basically **pipelining/batching** — decouple "how fast can I produce data" from "how fast can I deliver data" by adding a buffer stage. You see this pattern constantly in CS: network buffering, I/O batching, double-buffering in graphics. Same idea here, just applied to memory chip design.

## 5. Burst Transfers — Reading in Batches, Not Single Words

Since there's a fixed "setup cost" to open a row and prepare a transfer, DRAM lets you request multiple words at once (2, 4, 8) in a single "burst" rather than paying setup costs repeatedly for each individual word.

**The CS concept:** this is why **cache lines are wide** (64 bytes, not 4 or 8 bytes) instead of loading one word at a time. It's the amortization principle — pay a fixed overhead once, get multiple useful units of data out of it. Same reasoning as reading a whole disk block instead of one byte, or fetching a whole network packet instead of one byte at a time.

## 6. Timing Parameters (CL, tRCD, tRP, tRAS) — You Don't Need the Numbers, Just the Idea

The paper lists these DDR timing specs (CAS Latency, RAS-to-CAS delay, etc.). You don't need to memorize them. The one conceptual takeaway:

**Memory access latency isn't one number — it's a sum of several sequential delay stages**, and different stages get "hidden" or exposed depending on your access pattern. Sequential access hides most of these delays (row stays open); random access exposes almost all of them (row keeps closing/reopening). This is the hardware justification for why the paper later obsesses over access pattern optimization.

---

## The One-Sentence Summary of This Section

**Underneath the cache hierarchy you already know about, the RAM chips themselves have their own built-in "locality reward system"** (open rows = cheap, row switches = expensive) — meaning sequential-vs-random access isn't just a caching concern, it's baked into the physical memory technology itself, one level below where most programmers ever think to look.