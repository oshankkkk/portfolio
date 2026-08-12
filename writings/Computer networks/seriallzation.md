---
Title: Understanding serialization
date: 2026-07-20
---
##### Serialization and deserialization
Serialization is the process of converting an object or data structure into a sequence of bytes so it can be saved to a file, database or shared between processes or networks. Later, those bytes can be converted back into the original object. This reverse process is called deserialization.

All serialization produces bytes the difference is what those bytes represent.Text serialization is where the bytes represent human readable characters and binary serialization is where the bytes represent the data directly in a machine oriented format. It's called text serialization because the serialized bytes are valid text. If you built your own JSON-like format that doesn't represent text in UTF-8, that's binary serialization.

Serialization exists because objects only exist in your program's memory. Memory disappears when the program exits, and another program or computer can't directly understand your program's memory layout. Why can't you just send memory? Because memory is not portable. At the end of the day you work with memory using different address of where data is stored, and those addresses are places where data is stored at the time when the program runs.

Suppose `title` is a pointer with the value `0x7ffe12345678`. If you send those raw bytes to another computer, that address means absolutely nothing there pointers are only valid inside the process that created them. Serialization replaces pointers with the actual data they point to.

Serialization is used whenever data needs to leave your program's memory whether that's to be saved to disk, sent over a network, stored in a database, cached, or passed to another process. As soon as data crosses the boundary of your process's RAM, it almost always needs to be serialized first.

Not every sequence of bytes on disk is the result of serializing an object in your program. A random binary blob is just bytes. A compiled executable (.exe, ELF binary) is machine code and metadata, not a serialized C struct. But whenever you're taking structured data — objects, structs, arrays, maps — and storing or transmitting it, you're using some form of serialization.


