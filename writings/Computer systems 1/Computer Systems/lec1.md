# Lecture Notes: Bits, Bytes, and Integers
### Base 2 and bits
The term **"bit"** was coined around 1948 by Claude Shannon, founder of information theory, who defined it as the most primitive unit of information. (Shannon didn't invent binary computing itself.)
ENIAC, one of the first large scale computers, represented numbers in decimal, using 10 vacuum tubes per digit.

```
          Hundreds
Value: 0 1 2 3 4 5 6 7 8 9
Tube:  . . . . . ● . . . .

            Tens
Value: 0 1 2 3 4 5 6 7 8 9
Tube:  . ● . . . . . . . .

            Ones
Value: 0 1 2 3 4 5 6 7 8 9
Tube:  . . ● . . . . . . .
```

  Real electrical signals are noisy not clean square waves. Circuit components are small and imperfect. Representing only two states (0/1) gives you a large margin for noise/variation while still reliably distinguishing the two values. Storage circuits (e.g., feedback loops) can stably hold a 0 or a 1; more discrete levels are much more vulnerable to decay/noise/corruption.
  
  > A bit pattern has no inherent meaning*on its own. It isn't "a number" or "a string" or "an instruction" by itself meaning comes entirely from how we choose to interpret* it.
### Hexadecimal notation
Since writing out long bits is hard, so we group bits into 4-bit chunks (nibbles), each representing a value 0 to 15, written as a single hex digit, yk the base 16 numbers. C-style notation: prefix with `0x` (case-insensitive `x`), digits `0–9` then `A–F` (case-insensitive).
```
  Decimal: 15213 
  Binary nibbles: `0011 1011 0110 1101`
  Hex: `0x3B6D`
```

| Hex   | Binary | Dec | Hex | Binary | Dec |
| ----- | ------ | --- | --- | ------ | --- |
| 0     | 0000   | 0   | 8   | 1000   | 8   |
| 1     | 0001   | 1   | 9   | 1001   | 9   |
| 2     | 0010   | 2   | A   | 1010   | 10  |
| 3     | 0011   | 3   | B   | 1011   | 11  |
| 4     | 0100   | 4   | C   | 1100   | 12  |
| 5     | 0101   | 5   | D   | 1101   | 13  |
| 6     | 0110   | 6   | E   | 1110   | 14  |
| 7<br> | 0111   | 7   | F   | 1111   | 15  |

### Why bytes
Why not use individual bits?

A single bit can only represent two values (0 or 1). That's enough for a yes/no question, but not for most data. For example:

A decimal digit (0–9) needs at least 4 bits.
An English letter needs more than 4 bits.
Memory hardware works more efficiently when reading and writing groups of bits rather than one bit at a time.
Why 8 bits?

Historically, computers used different byte sizes (6, 7, 8, 9 bits, etc.). Over time, 8 bits became the standard because:

It can represent 256 different values (2⁸ = 256), which is enough for:
integers from 0 to 255
one ASCII character (originally 7 bits, with room for an extra bit)
It is a good balance between storage efficiency and hardware simplicity.
Modern computer architectures, memory, and communication standards are built around 8-bit bytes.
### 64 vs 32 bit

#### What is a word
In computing, a word is the natural chunk of data size a CPU is designed to process at once basically the size of the CPU's registers and the width of its main data pathways.

- On a 32-bit CPU, a word is 32 bits (4 bytes)
- On a 64-bit CPU, a word is 64 bits (8 bytes)

Almost all modern CPUs (Intel/AMD) are 64-bit capable. Its the ceiling of what the chip is able to run not how it runs things all the time.
Register size of the CPU is also same as the word size, same goes to the address/pointer size. since the pointers range is hight in 64 bit, it better and that why ppl started using it.

An OS is installed as either a 32-bit build or a 64-bit build. You can install a 32bit one on the 64bit one as well, but it will be wastefull.

- If the OS is 32-bit → it can only run 32-bit programs, period.
==If the OS is 64-bit → it can run 64-bit programs and thanks to "compatibility mode" (e.g. Windows' WoW64 — "Windows on Windows 64", what bout linux and unix??), it can also run older 32-bit programs by emulating the needed 32-bit environment.==


Even on a 64-bit OS with a 64-bit CPU, a compiler doesn't automatically produce 64-bit machine code. That's decided by:

- **Which compiler/toolchain** you use, and what its default target architecture is
- **Compile-time flags** you pass it, e.g.:
  - GCC/Clang: `-m32` vs `-m64`
  - MSVC: choosing the `x86` vs `x64` build configuration
  - Rust: `--target i686-pc-windows-msvc` vs `x86_64-pc-windows-msvc`
  
==whats the default.==

So you could take identical source code, compile it with `-m32`, and get a 32-bit executable even while sitting on a fully 64-bit OS and CPU (which will run cause of compatilbilty mode). 
#### Boolean algebra to digital logic
George Boole (19th century) formalized propositional logic (true/false reasoning) as an algebra. Claude Shannon connected Boolean algebra to the design of circuits originally built from electromechanical relays, stuff like telephone switching networks not transistors. Before this connection was made, such circuits were designed without a systematic algebraic method.

> Another rabbit hole, anyway claude shannon used boolean algebra to make logic gates.
#### Basic boolean operators
| Logic Gate         | Description                                      |
| ------------------ | ------------------------------------------------ |
| AND                | True only if both inputs are true.               |
| OR (inclusive)     | True if either or both inputs are true.          |
| NOT                | Flips true/false.                                |
| XOR (exclusive OR) | True if exactly one input is true, not both.<br> |
#### Bit vectors

![[lec1-1786026890138.webp]]

Above are Set A = {A, C, E}, Set B = {B, C, D}.

- AND only lit up at C, because C is the only element present in both sets (set intersection).
- OR lit up everywhere, because every element shows up in at least one set (that's the union).
- XOR lit up everywhere except C, because C is the one element that's in both, and XOR is only true when exactly one input is true (symmetric difference).
#### Practical example 
Real-world use: e.g., tracking which of many network connections are active/being listened to (bit masks). Libraries exist to manipulate these sets, but underneath they're just bitwise operations on words.
A bit pattern used to select and ignore certain positions is often called a mask (1 = "care about this bit," 0 = "ignore it"), used with AND to isolate bits and stuff.
Also the stone representation theorum relates to this in someway i think.
##### Bitwise and logical ops
| Operator | Type    | Description                       |
| -------- | ------- | --------------------------------- |
| `&`      | Bitwise | AND each bit                      |
| `\|`     | Bitwise | OR each bit                       |
| `^`      | Bitwise | XOR each bit                      |
| `~`      | Bitwise | NOT (invert bits)                 |
| `&&`     | Logical | AND truth values (short-circuits) |
| `\|\|`   | Logical | OR truth values (short-circuits)  |
| `!`      | Logical | NOT truth value                   |
#### Unsigned vs. Two's Complement
Two major classes of integers in this course:
- Unsigned: values ≥ 0 only.
- Signed: can be positive or negative. Many encodings are possible in principle, but the overwhelmingly standard one (and the one used on essentially all modern machines) is  two's complement.
#### Bit weighting
- **Unsigned**: ordinary binary every bit `i` contributes $+2^i$.
- **Two's complement**: identical, **except** the most-significant (leftmost) bit contributes a **negative** weight, $-2^{w-1}$ (where *w* = word size in bits). All other bits keep their normal positive weight.
#### Worked examples (5-bit words, illustrative)
- `10` (positive): sign bit = 0; ordinary positive weighting.
- `-10`: start from the sign bit's weight $-2^{4} = -16$ (most negative available), then **add back** positive powers of 2 to reach the desired value, e.g., $-16 + 4 + 2 = -10$.
- General mental model for two's complement: **"start very negative, add back positive powers of two to reach your target value."**

### Rule to Remember

- If the most significant bit (MSB) is **0**, the signed and unsigned values are the same.
- If the MSB is **1**:
  - Unsigned interprets it as a large positive number.
  - Signed (two's complement) interprets it as a negative number.
- **The bits never change—only their meaning does.**
### 16-bit example
- $15213$ or $-15213$ representable by choosing the sign bit's weight of $2^{15} = 32768$ appropriately, plus the remaining lower bits.

### Range of values

|                                | Smallest                               | Largest                                        |
| ------------------------------ | -------------------------------------- | ---------------------------------------------- |
| **Unsigned**, *w* bits         | $0$                                    | $2^w - 1$ (all bits = 1)                       |
| **Two's complement**, *w* bits | $-2^{w-1}$ (sign bit only = 1, `TMin`) | $2^{w-1} - 1$ (sign bit = 0, rest = 1, `TMax`) |

- **All-ones bit pattern** = `-1` in two's complement (sign bit "very negative," plus all positive bits "add back" to exactly cancel to $-1$).
- **All-zeros bit pattern** = `0` in either representation.
- **Rule of thumb**: a hex value that's mostly `F`s (e.g., `0xFFFF...`) is very likely a **negative** two's complement number.
- **Asymmetry**: the most-negative two's complement value has **no positive counterpart** of equal magnitude (e.g., for 4 bits: range is $-8$ to $7$, not $-8$ to $8$). This is because the encoding must also reserve a pattern for **zero** — there's no "extra" pattern to give the positive side.

Unsigned can represent roughly twice the magnitude of positive values that two's complement can, since unsigned devotes all bits to magnitude.

>C does not guarantee two's complement or any particular ranges are portable across all possible machines  use `<limits.h>` (defines `INT_MAX`, `INT_MIN`, `UINT_MAX`, etc.) for machine/compiler-correct bounds rather than hardcoding assumptions (though many programmers casually assume standard values anyway).

| Binary | Decimal | Binary | Decimal |
| ------ | ------: | ------ | ------: |
| `1000` |      -8 | `0000` |       0 |
| `1001` |      -7 | `0001` |       1 |
| `1010` |      -6 | `0010` |       2 |
| `1011` |      -5 | `0011` |       3 |
| `1100` |      -4 | `0100` |       4 |
| `1101` |      -3 | `0101` |       5 |
| `1110` |      -2 | `0110` |       6 |
| `1111` |      -1 | `0111` |       7 |

---
## 9. Casting Between Signed and Unsigned in C

- **Key fact**: casting between signed (`int`) and unsigned (`unsigned int`) in C **does not change the underlying bits at all**. Only the *interpretation* of the high-order bit's weight (positive vs. negative) changes.
  - This is fundamentally different from casting **float → int**, where the actual bit pattern *does* change (covered next week).
- Example: `int x = -1; unsigned u = (unsigned) x;`
  - Common wrong guess: "should give 0, off by one, partial credit-worthy." **Not what happens.**
  - Correct answer: `u` becomes **`UINT_MAX`**, i.e., $2^{32} - 1$ (all bits = 1), because `-1` is *already* the all-ones bit pattern — the cast just reinterprets those same bits as a large positive unsigned number.
- This bit-preserving reinterpretation applies to **explicit casts** and also to **implicit casts** — e.g., assigning an `unsigned` value to a signed `int` variable, or vice versa, or passing/returning values of mismatched signedness through functions — all trigger this silent reinterpretation without any obvious syntax marking it.

### Implicit conversion rule in mixed-signedness expressions
- **C rule**: if an operation (arithmetic *or* comparison, e.g. `<`, `>`) has one signed and one unsigned operand, the **signed operand is implicitly converted to unsigned** before the operation proceeds.
- **Literal suffix note**: an integer literal like `0` is signed by default; appending `u`/`U` (e.g., `0u`) makes it unsigned.

### Surprising comparison examples
| Expression                                                                       | Result                        | Why                                                                                                            |
| -------------------------------------------------------------------------------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `0 == 0u`                                                                        | true                          | Both are all-zero bits; near zero, signed/unsigned agree.                                                      |
| `-1 < 0`                                                                         | true                          | Ordinary signed comparison.                                                                                    |
| `-1 < 0u`                                                                        | **false** (`-1` is "greater") | `-1` is implicitly cast to unsigned → becomes `UINT_MAX`, a huge positive number → greater than `0`.           |
| `TMax > TMin` (both signed)                                                      | true                          | Ordinary signed comparison: positive > negative.                                                               |
| `TMax > (unsigned) TMin`                                                         | **false**                     | `TMin`'s bit pattern (`1000...0`) reinterpreted as unsigned is a **huge** positive number, larger than `TMax`. |
| `(unsigned)(0 followed by all 1s)` vs `(1 followed by all 0s)`, unsigned compare | first is smaller              | Straightforward unsigned magnitude comparison.                                                                 |
| Same patterns, **signed** compare                                                | first is **larger**           | First pattern (sign bit 0) is positive; second (sign bit 1) is negative.                                       |

---

#### Real Bugs Caused by Unsigned Arithmetic

```c
// example 1
unsigned i;
//Problem is that `i` is `unsigned`, so `i >= 0` is always true infinite loop (logically). When `i` reaches 0 and is decremented, it doesn't go negative it wraps around to `UINT_MAX` (all ones).
for (i = length - 1; i >= 0; i--) {
    // sum array backward
}

// example 2
//size_t is always unsigned.
size_t delta = sizeof(...);   // some positive constant
int i;
...
//Even though `i` starts out as a signed `int`, subtracting an unsigned `delta` from it implicitly converts the whole expression to unsigned 
//`i - delta` is  never interpreted as negative, so the loop again never terminates as intended 
while (i - delta >= 0) {  // intent: stop once close to zero
    ...
}
```
#### Sign Extension (Expanding Bit Width, Signed) Truncation (Reducing Bit Width, Signed)
Converting a signed value from a smaller word size to a larger one while preserving its numeric value by replicate the original sign bit (the leftmost/most-significant bit) into all the new, wider positions.

![[lec1-1786103268411.webp]]

![[lec1-1786103333209.webp]]

#### Truncation
Going from a larger representation to a smaller one discards information. Mechanism simply drop the extra high-order bit(s). Whatever bit becomes the new most-significant bit takes on the new (smaller) representation's sign-bit weight. Truncation is safe only when the value already fits within the smaller type's range

![[lec1-1786103700356.webp]]

![[lec1-1786103805694.webp]]