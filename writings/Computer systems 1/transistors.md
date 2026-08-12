---
id: transistors
aliases: []
tags: []
---

Transistors combine to make logic gates. not gate is the most basic one, after that we combine them to make AND and OR and combine them to make XOR and NOR.  
We use these logic gates to make a adder and use adders to make 8 bit adders.
We also make decorders which given a input match it to a output
We combine all of them to make the ALU.

Then we make memory, we take a logic gate and connect its output to its own input, this way we can lock the logic gate in either 1 or 0 state forever, regardless of the input.
OR is locked in 1 and AND locks in 0. Now we combine the 2 to make a circuit which we can make 1 or 0 and lock the output, essentially making the circuit remember its value. 
This is called a AND OR latch and it can store 1 bit of memory.
A Gated latch is a better way of doing this, there are even more ways of doing this btw.
A gated latch has 2 inputs, a data and a write enable. if write is enabled the data is remembered, else its ignored.
A 8 bit register is made up of 8 of these, they have 1 common write enable to reset the whole 8 of these as well.

So we make this collection of latch into a grid, and we can find each latch with a coordinate. This is that memory bits address.

This is for 1 bit, this is enlarged for 8 bytes and thats how memory works.

Binary decorder and the address bus and how 32 byts systems can handle only upto 4gb 


Registers are made with SRAM like transistors
This memory is called SRAM, too many transistors.

DRAM is when we have capacitors to store the bit values. When you check a 1 capacitors value, the capacitor free the voltes turn it into a 0. Also it has come problems with how the grid and the decorder works so its made in a way to find the voltage above a threshold to get the value. 
