# Lecture Notes: Integer Representations, Arithmetic, and Memory Layout

>If an operation (e.g. addition, less-than) has one signed and one unsigned operand, C's rule is: the signed value is implicitly converted to unsigned, and the operation is then performed using unsigned arithmetic. This is a common source of subtle bugs.
## 1. Unsigned addition wraps around

Add `200 + 100` using 8 bits:

```
  200 = 11001000
+ 100 = 01100100
-----------------
  300 = 100101100   (9 bits needed for the true sum)
```

Only 8 bits exist, so the leading `1` (worth 256) gets dropped:

```
00101100 = 44
```

And indeed `300 - 256 = 44`. That's the whole rule: **unsigned overflow = subtract 2^w**. C guarantees this behavior — it's not a bug, it's modular arithmetic by design.

## 2. Two's complement uses the *same* circuit

Take those exact same bits — `11001000` and `01100100` — but now read them as **signed**:

- `11001000` as signed = **-56**
- `01100100` as signed = **100**

The adder doesn't know or care about signedness. It just adds bits and produces `00101100` again. Read as signed, that's **+44** — which is exactly `-56 + 100 = 44`. ✅ No reinterpretation needed, no overflow here (opposite signs never overflow — more on that below).

## 3. Two kinds of signed overflow

**Positive overflow** (two positives, sum too big):
```
100 + 100 = 200 (true sum)
```
`200` as an 8-bit signed pattern (`11001000`) reads as **-56**. Check: `200 - 256 = -56`. The result silently flips negative even though you added two positive numbers.

**Negative overflow** (two negatives, sum too negative):
```
-100 + -100 = -200 (true sum)
```
As bits, this becomes `00111000` = **+56**. Check: `-200 + 256 = 56`. Two negatives just produced a positive result.

**Why opposite signs are always safe:** the true sum of a positive and a negative always lands *between* the two operands — and both operands were already representable, so the sum is too. That's why my §2 example (-56 + 100) never overflowed.

## 4. Multiplication truncates the same way

`200 × 200 = 40000` (true product, needs way more than 8 bits). Keep only the low 8 bits: `40000 mod 256 = 64`.

Now do it signed: `200` as signed is `-56`, and `(-56) × (-56) = 3136`. Low 8 bits: `3136 mod 256 = 64` — **the exact same bit pattern.**

This is why hardware needs only *one* multiply instruction for both signed and unsigned — the low bits never differ. You only need a special instruction if you want the *full* untruncated product (e.g., x86's widening multiply).

## 5. Left shift = multiply by a power of 2

```
5 << 3 = 40      (5 × 8)
5 << 5 = 160     (5 × 32)
160 - 40 = 120   (5 × 24)
```
So a compiler can turn `x * 24` into `(x<<5) - (x<<3)`, since multiplication distributes over subtraction even under truncation.

## 6. Right shift = divide by a power of 2 — but only cleanly for unsigned

Unsigned, logical shift (fills with 0s): `29 >> 2` → `00011101 >> 2 = 00000111 = 7`. Matches `floor(29/4) = 7`. ✅

Signed, negative number, arithmetic shift (fills with the **sign bit**):
```
-29 = 11100011
-29 >> 2 (arithmetic) = 11111000 = -8
```
But true division rounds toward zero: `-29/4 = -7.25 → -7`, not -8. The arithmetic shift rounded toward *negative infinity* instead — off by one.

## 7. The bias trick fixes it

Add `2^k - 1` (here `2^2 - 1 = 3`) **before** shifting a negative number:

```
-29 + 3 = -26 = 11100110
-26 >> 2 (arithmetic) = 11111001 = -7   ✅ correct!
```
If the low bits being shifted out were already all zero (evenly divisible), adding this bias changes nothing. If they weren't, the bias carries into the kept bits and nudges the result toward zero — exactly compensating for the arithmetic shift's bias toward -∞.

## 8. Negation: flip the bits, add 1

```
x = 5 = 00000101
~x    = 11111010  = -6
~x + 1= 11111011  = -5   ✅
```
Why it works: `x + ~x` is always all-ones (`-1`), so `~x = -1 - x`, and `~x + 1 = -x`.

## 9. The two special cases (always test these first!)

- **x = 0:** `~0 = 11111111 = -1`, then `+1 = 00000000 = 0`. So `-0 = 0`. ✅ fine.
- **x = TMin (-128):** `~(10000000) = 01111111 = 127 (TMax)`, then `+1 = 10000000` — that overflows right back to **-128**. So `-TMin = TMin`. Negating the most negative number gives you... the same number. This is the classic gotcha that breaks naive `abs()` functions.

## 10. Safe unsigned countdown loop

```c
size_t n = 10;
for (size_t i = n - 1; i < n; i--) {
    // use i
}
```
Once `i` would go below 0, unsigned wraparound (guaranteed by C) sends it to `SIZE_MAX` — a huge number — which fails the `i < n` test and stops the loop. No need to avoid unsigned indices; just don't test `i >= 0` (always true for unsigned, so it'd loop forever).


Want me to turn any of these into a quick set of practice problems you can work through yourself?
### Other Legitimate Uses of Unsigned Numbers

- Unsigned arithmetic has a fully specified behavior even on overflow (modular arithmetic), which is useful for multiprecision arithmetic.
- Useful for representing sets of bit flags (no concept of "negative" needed).
- Common in systems programming for representing addresses, masks, and similar bit collections.
- Note: Java deliberately does not include unsigned integer types, specifically to avoid these kinds of quirks. C, by contrast, has unsigned types built in and uses them extensively.

### True/False Quiz Walkthrough (Rapid-Fire, with Counterexamples)

General strategy emphasized throughout: to disprove a claimed-always-true statement, try x = TMin (or occasionally x = 0) as a counterexample first.

1. If x < 0, is 2*x < 0 always true? No. Counterexample: x = TMin. Since multiplying by 2 is a left shift, shifting TMin left by 1 drops the sign bit and the result becomes 0 (not negative).
2. Is an unsigned value ux always >= 0? Yes - by definition, unsigned values cannot be negative.
3. For a 32-bit int x, if (x & 7) == 7 (i.e., the low 3 bits of x are all 1), is (x << 30) guaranteed negative? Yes. Since the low 3 bits are all ones, shifting left by 30 moves two of those one-bits into the top two bit positions, including the sign bit (bit 31), guaranteeing a negative result.
4. Is ux > -1 (as an unsigned comparison) ever true? No, essentially never. Reasoning: comparing an unsigned value against a signed constant like -1 triggers the C rule that promotes the signed value to unsigned; -1 as unsigned becomes UMax (all ones, the largest possible unsigned value), so ux > UMax can never be true for any unsigned ux.
5. If x > y, is -x < -y always true? No. Counterexample: y = TMin. Negating TMin gives TMin back (self-negation edge case), and no value is less than TMin, so the implication fails.
6. Is x * x always >= 0? No. Counterexample from an earlier lecture: squaring certain large values can overflow and produce a value that isn't >= 0 (e.g., a specific large value's square, cited as not being nonnegative due to overflow).
7. If x and y are both positive, is x + y always positive? No - can overflow to a negative result (positive overflow case).
8. If x >= 0, is -x <= 0 always true? Yes, this one holds.
9. If x <= 0, is -x >= 0 always true? No. Counterexample: x = TMin (same self-negation edge case as above).
10. If you compute (x & -x), then arithmetic-shift the result right by 31 (on a 32-bit machine), is the result always equal to -1? No (nearly always true, but not always). Counterexample: x = 0.
11. Is shifting an unsigned number right by 3 the same as dividing it by 8? Yes, for unsigned numbers, logical right shift by k is exactly equivalent to integer division by 2^k.
12. Is the same true for a signed number (arithmetic right shift by 3 equals division by 8)? No - as discussed above, arithmetic right shift rounds toward negative infinity for negative values, whereas C integer division rounds toward zero, so they disagree for negative operands.

## Memory Layout and Addressing
- At the assembly/machine-code level, programs reference memory via addresses. Conceptually, memory is just virtual memory: a large array of bytes spanning a wide range of addresses.
- Physically, virtual memory is implemented via a combination of hardware and operating system software, using caches, RAM, disk/solid-state storage, and potentially network resources, all working together to make it appear as one large, uniform memory space.
- Each running program (a process) has its own separate virtual address space. This isolation means a bug or misbehavior in one program cannot directly corrupt another program's memory. Multiple processes can be running the same program simultaneously, each with its own virtual address space.
- Word size: there's no single precise definition, but broadly it refers to the size of addresses a machine can handle, which is a function of both the hardware and how the code was compiled (32-bit vs 64-bit mode).
  - A 32-bit machine/mode: addresses up to 2^32 - 1, roughly 4 gigabytes of addressable space.
  - A 64-bit machine/mode: in principle addresses up to 2^64, an enormously larger range (order of 10^19 bytes, i.e., exabytes) - far beyond what any realistic storage system could hold (a single machine would need roughly a thousand large disk drives to have that much actual storage).
  - In practice, real 64-bit hardware today implements only the first 48 bits of address space, rather than the full 64 bits, since the full range is unnecessary.
- Multi-byte quantities: a value's address is defined as the address of its lowest-numbered byte. E.g., a 32-bit (4-byte) quantity's address is the address of its lowest byte.
- Data types can change size depending on execution mode - notably, a pointer/address is 4 bytes in 32-bit mode and 8 bytes in 64-bit mode.
### Byte Ordering: Big-Endian vs Little-Endian

- When a multi-byte value is stored in memory, there are two natural conventions for byte order: most-significant byte first, or least-significant byte first. This is called byte ordering, commonly known as big-endian versus little-endian.
- The terminology originates as a reference to Gulliver's Travels, in which characters fought over which end of a boiled egg to open - used metaphorically for the historical arguments over byte-ordering conventions (even though the book predates computers).
- x86 hardware - and therefore Windows, macOS, and Linux running on x86 - uses little-endian ordering: least-significant byte stored at the lowest address.
- ARM processors (common in phones) can support either ordering in hardware, but under common operating systems they are run in little-endian mode.
- Internet protocols specify big-endian byte order ("network byte order"). This means code that sends/receives data over a network on a little-endian machine must explicitly convert byte order using special functions.
- Example: the 4-byte hex value written as 0x01020304 (most significant byte on the left, by normal written convention) would appear "in order" (leftmost byte at lowest address) on a big-endian machine, but appears reversed in memory order on a little-endian machine.
- Historical example given: older Sun Microsystems machines (later acquired by Oracle) were big-endian; such machines are now uncommon, especially outside specialized systems like some database servers.
- Practical example: the value 15213 has a particular hex byte representation; on a little-endian (x86) machine, the least-significant byte appears first in memory; on a big-endian (Sun) machine, the same bytes would appear in reverse order.
- Course lab material (referenced) includes example code that prints out the raw byte-by-byte memory representation of different data types, useful for directly observing endianness on your own machine.
- Pointers are just addresses (essentially indices into the virtual memory array). Their actual values depend on the machine, operating system, and compiler, and can vary from run to run - there is no fixed, portable representation.
- Strings in C are always represented as a sequence of single-byte (ASCII) values terminated by a zero byte. This representation is independent of byte ordering, since it's processed one byte at a time. Because of this endian-independence, cross-platform software sometimes encodes numeric data as strings (rather than raw binary) specifically to avoid byte-ordering compatibility issues.
- Relevance to the upcoming lab: disassembled x86 machine code often displays multi-byte numeric values with what looks like reversed byte order in the printed hex - not truly "backwards," but rather byte-pair order reversed relative to normal written convention. Example given: a value normally written as 0x0102ab in the disassembly might display as ab1202 or similar reversed-byte form (illustrative example: "one two ab" becomes "ab12").
