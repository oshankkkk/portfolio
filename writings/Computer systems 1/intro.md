---
id: intro
aliases: []
tags: []
Title: Tour of computer systems
date: 2026-08-09
---
```c
#include <stdio.h>
int main(){
printf("helo world");
return 0;
}
```

This is ultimatly a textfile. The letters are all ASCII characters, they are represented in 8 bit values (each character here is 1 byte). 

>"All information in a system—including disk files, programs stored in memory, user data stored in memory, and data transferred across a network—is represented as a bunch of bits. The only thing that distinguishes different data objects is the context in which we view them. For example, in different contexts, the same sequence of bytes might represent an integer, floating-point number, character string, or machine instruction."

![[n1-1786262943263.webp|769x196]]

- The preprossesor processes the header files and puts them into the callers c file.
- The compiler compiles the whole thing into assemebly which is basically machine code but in text.
- The assemebler converts the assembly into machine code.  
- The linker merges the library functions to our c code.

> The lore behind the GNU (Gnu's Not Unix) project is very interesting and worth checking out.

![[n1-1786265726478.webp|672x350]]

The bus is a channel which passes data inside the cpu. They are designed to pass chucks of bits. Usually a chuck could be either 32bits or 64bits(in morden cpus).
I/O devices are connected to the cpu through a controller or a adapter. A controller is a build it chip(silicon cricuit)that acts as a interface for the I/O device, while the adapter is somthing you plugin.
Memory is a collecetion of DRAM chips. Logically is a array of bytes.  

> Dunno what DRAM is 

The program counter is a register with the capacity of the word size. The processor operates according to the instruction set architecture. Basically the pc points to a instruction in memory, the cpu reads the instructions and perform some operation, then pc jump to the next instruction and the cycle continues.These are some basic instrucitons that are related for the ALU,memory and the register file(if you made somthing like a VM for a interpreter, you will know what i mean).The register file is a collection of register with the cap of the word size and the ALU does the calculation stuff.

> There ISA and microarchitectures i dont know bout

When we type something it goes to the cpu registers and it goes to the memory. There is a technique called direct memory access that sends data directly to the main memory and not through the registers.
The CPU does a lot of moving of data from and to different parts of the system for it to complete the actual task. The book calls this is as "overhead" and removing it is goal of optimisation. Its easier to make faster CPUs than memory so the gap between processor to memory is increasing. To solve this we add more caches inbetween them. 

The L1 cache is inside the processor chip and is the same speed as the internal cpu registers. The L2 cache is outside, connected to the CPU via a special bus. L2 cache is like 5 times slower than L1 according to the book. But its still faster than accessing the main memory. The L1 and 2 caches are implemented with SRAM tech (not DRAM, like the main memory). Using caches improves the performance cause of this temporial and spatial locally thing, which is basiclly that programs tend to access prevoiusly used data or data around that so we store those in the cache, hence increasing performace.

> Most morden CPUs now have a L3 cache as well.

![[n1-1786287277805.webp]]

> https://youtu.be/7yrK_9PderQ?si=xTpsm5C_IXKm5cI5 bitlemon on memory hierachy 

The OS manages the hardware for application to run and they use the provided uniform OS abstractions to access hardware functions. This also makes the system secure from bad application software.
A very important OS level abstraction is a process. Each program is isolated in its own process by the OS. Concurrent process execution is achieved by switch amoung processes.
The OS tracks all the info on the states the processes (like the memory pointers and register file stuff) uses and its called context, when the processor switch to another process it stores the current processes context and reloads the other one. This is called context switching (duhh).

We made threads cause processes are not small enough, we need multiple instances of the same process so we made threads, thats basically like process inside a process. This means you can have multiple insteances of the same thing in threads in 1 process.

![[n1-1786292879004.webp|720x396]]

A file is a sequence of bytes, and in unix everything every I/O device, including disks, keyboards, displays, and even networks is treated like a file. All input and output in the system is performed by reading and writing files, using a small set of system calls known as Unix I/O.

>"This simple and elegant notion of a file is nonetheless very powerful because it provides applications with a uniform view of all the varied I/O devices that might be contained in the system. For example, application programmers who manipulate the contents of a disk file are blissfully unaware of the specific disk technology"

Then there is a small intro in networking which doesnt have that much to it.

uniprocessor = single core, multiprocssor = multicore 

![[n1-1786295810537.webp|639x518]]

Hyper threading is like core level multithreading (so its dual threading), like each core is divided into 2 logical cores and work is done in parallel.

In the thread a process what stuff do they share and have to there own?

- Multicore parallelism = 2 cores, 2 independent instruction streams, physically separate hardware — genuinely two things happening in two different places at once.
- ILP = 1 core, 1 instruction stream, but the core's internal machinery (pipeline stages, multiple execution units, out-of-order scheduling) finds independent instructions inside that single stream and overlaps or co-executes them.
- SMT/hyperthreading = 1 core, but 2 instruction streams (threads) sharing that core's execution units, filling in gaps that ILP alone couldn't fill from just one thread.

![[n1-1786303512037.webp|445]]


