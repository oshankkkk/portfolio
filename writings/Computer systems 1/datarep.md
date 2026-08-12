---
id: datarep
aliases: []
tags: []
---
So computers work with bits and bytes. We use them to represent stuff. When we represent intergers we reperesent signed vs unsigned its. 
See 5 in binary is 00000101 and if we convert it to -5 in binary its,

```
 
 00000101
 ↓↓↓↓↓↓↓↓
 11111010
 
 11111010
 +       1
 ---------
 11111011
 
 +5 = 00000101
 -5 = 11111011
 
 bits:    1  1  1  1  1  0  1  1
 weights: -128 64 32 16 8 4 2 1
 
 -128 + 64 + 32 + 16 + 8 + 0 + 2 + 1
 = -5

```

Signed bits wrap around

```
 
 0000 =  0
 0001 =  1
 0010 =  2
 0011 =  3
 0100 =  4
 0101 =  5
 0110 =  6
 0111 =  7
 1000 = -8
 1001 = -7
 1010 = -6
 ...
 1111 = -1
 
 0111  =  7
 +0001
 ------
 1000  = -8
 
```

So the signed 8 bits go from 0 to -1 and peaks at 127 and then its -128 and goes down from there

```

 00000000 =   0
 00000001 =   1
 ...
 01111111 = 127   ← maximum
 10000000 = -128  ← wraps around
 10000001 = -127
 ...
 11111110 = -2
 11111111 = -1

```

Uint and int errors in C is undefined.

There is also overflows that i need to understand, this comes in flowing point errors.
