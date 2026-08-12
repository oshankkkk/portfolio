---
date: 2026-07-26
Title:
tags: []
---
System design is the process of designing the architecture of a software system: deciding what components exist, what responsibilities they have, how they communicate, and how the system handles growth, failures, and data.

Prolly try to Designing Data-Intensive Applications

> System design. There are two paths ahead of you. Path number one is to study all the theoretical concepts about system design. You can learn all the trade-offs about different databases, CDNs, replicas, and related concepts. Or you could just go build something. You could build enough systems and develop a baseline intuition for how things should be designed, so you don't have to memorize specific solutions. You don't need to think, "What if they ask me to design a Twitter clone? Let me memorize how to build a Twitter clone." Instead, you can create solutions from your own understanding. That is the approach I optimized for. I did some research into the kinds of things usually asked in system design interviews, and then I thought through how I would build them. If they asked me to design a Twitter clone, YouTube, or Dropbox, how would I approach the architecture? This is where knowing the platform you're building on becomes really useful. For example, during an interview, I wanted to use rsync for one of the system components. One of the requirements involved strict speed and bandwidth constraints, and the interviewer asked whether rsync would copy the entire file to perform a diff. I confidently answered no—rsync uses checksums to compare files rather than copying the whole file. I knew that because I understood how the technology worked. 

Producer consumer method
