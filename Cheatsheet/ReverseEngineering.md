# Reverse Engineering CTF Cheatsheet

A practical cheatsheet for solving beginner/intermediate **Reverse Engineering (RE)** CTF challenges.

The goal is not to understand every instruction in a binary.

The goal is:

```text
INPUT
  ↓
PROCESSING
  ↓
VALIDATION
  ↓
SUCCESS / FAILURE
```

Find that path.

---

# 1. First Steps

## Identify the Binary

```bash
file ./chall
```

Example:

```text
ELF 64-bit LSB pie executable, x86-64
```

Important things to note:

| Property           | Why it matters                          |
| ------------------ | --------------------------------------- |
| ELF                | Linux executable                        |
| 64-bit             | x86-64 registers/instructions           |
| PIE                | Addresses change at runtime             |
| dynamically linked | Uses shared libraries                   |
| stripped           | Function names may be unavailable       |
| not stripped       | Symbols/function names may be available |

---

## Check Security Protections

If available:

```bash
checksec --file=./chall
```

Typical output:

```text
RELRO    STACK CANARY    NX    PIE
```

Quick interpretation:

```text
NX          → stack is not executable
PIE         → executable loads at random address
Canary      → stack overflow protection
RELRO       → GOT protection
```

For pure reverse engineering, these protections aren't always important, but they become very important when the challenge turns into exploitation.

---

# 2. Run the Program

Always run it first.

```bash
./chall
```

Try:

```text
normal input
empty input
wrong input
very long input
numbers
letters
special characters
```

Record:

* prompts
* expected input format
* success messages
* failure messages
* crashes
* number of inputs
* anything suspicious

---

# 3. Search Strings

Start with:

```bash
strings -a ./chall
```

More useful:

```bash
strings -a -n 4 ./chall
```

Search for interesting strings:

```bash
strings -a ./chall | grep -iE 'flag|pass|password|key|secret|win|correct|wrong|success|fail'
```

Also search for common CTF flag formats:

```bash
strings -a ./chall | grep -E 'HTB\{|NADI\{|FLAG\{|CTF\{'
```

### Why?

Strings can reveal:

```text
flag messages
debug messages
hardcoded passwords
file names
function hints
URLs
error messages
format strings
```

But remember:

> Not finding the flag in `strings` does NOT mean the flag isn't there.

It may be constructed dynamically.

---

# 4. Check Symbols

If the binary isn't stripped:

```bash
nm -C ./chall
```

Look for:

```text
main
win
flag
check
verify
validate
decrypt
encrypt
```

Filter:

```bash
nm -C ./chall | grep -Ei 'main|flag|win|check|verify|valid|decrypt'
```

In GDB:

```gdb
info functions
```

---

# 5. Useful ELF Tools

## `readelf`

Headers:

```bash
readelf -h ./chall
```

Sections:

```bash
readelf -S ./chall
```

Symbols:

```bash
readelf -sW ./chall
```

Dynamic dependencies:

```bash
readelf -d ./chall
```

Imported functions:

```bash
readelf -Ws ./chall | grep UND
```

---

## `objdump`

Disassemble:

```bash
objdump -d ./chall
```

Intel syntax:

```bash
objdump -d -M intel ./chall
```

Save to a file:

```bash
objdump -d -M intel ./chall > chall.asm
```

Search:

```bash
grep -n '<main>' chall.asm
```

---

# 6. GDB Basics

Start:

```bash
gdb -q ./chall
```

Useful setup:

```gdb
set disassembly-flavor intel
set pagination off
```

---

## Find Functions

```gdb
info functions
```

---

## Disassemble

```gdb
disassemble main
```

Specific function:

```gdb
disassemble function_name
```

Intel syntax:

```gdb
set disassembly-flavor intel
```

---

# 7. Running in GDB

Run:

```gdb
run
```

Pass arguments:

```gdb
run argument1 argument2
```

Start and stop at `main`:

```gdb
start
```

Continue:

```gdb
continue
```

or:

```gdb
c
```

---

# 8. Breakpoints

Break at a function:

```gdb
break main
```

Short form:

```gdb
b main
```

Break at an address:

```gdb
break *0x1234
```

List breakpoints:

```gdb
info breakpoints
```

Delete:

```gdb
delete 1
```

---

# 9. Stepping

Execute one instruction:

```gdb
si
```

Step over function calls:

```gdb
ni
```

Continue execution:

```gdb
c
```

### Difference

```text
si = step into
ni = step over
```

Usually:

```text
ni
```

is easier when exploring normal program logic.

---

# 10. Registers

Display registers:

```gdb
info registers
```

Individual register:

```gdb
p $rax
```

Hex:

```gdb
p/x $rax
```

Common x86-64 registers:

```text
RAX → return value / general purpose
RBX → general purpose
RCX → general purpose
RDX → argument / general purpose
RSI → argument
RDI → argument
RBP → stack frame
RSP → stack pointer
RIP → instruction pointer
```

---

# 11. The x86-64 Calling Convention

For Linux x86-64, the first arguments to a function are generally:

```text
1st → RDI
2nd → RSI
3rd → RDX
4th → RCX
5th → R8
6th → R9
```

Return value:

```text
RAX
```

Example:

```c
printf("Hello %d", x);
```

At the call to `printf`:

```text
RDI → format string
RSI → x
```

This is extremely useful when reversing function calls.

---

# 12. Examining Memory

The GDB `x` command means "examine memory."

Basic:

```gdb
x ADDRESS
```

Examples:

```gdb
x/10x ADDRESS
x/10d ADDRESS
x/10i ADDRESS
x/s ADDRESS
```

Format:

```text
x / COUNT FORMAT ADDRESS
```

Common formats:

```text
x → hexadecimal
d → decimal
u → unsigned decimal
s → string
i → instruction
c → character
```

Examples:

```gdb
x/10wx &array
```

Display 10 words in hex.

```gdb
x/20s ADDRESS
```

Display strings.

```gdb
x/20i $rip
```

Display instructions around the current instruction.

---

# 13. Inspecting Variables

If symbols are available:

```gdb
p variable
```

Hex:

```gdb
p/x variable
```

Arrays:

```gdb
p array
```

Memory:

```gdb
x/20wx &array
```

---

# 14. Understanding Common Assembly

## `mov`

```asm
mov rax, rbx
```

Conceptually:

```c
rax = rbx;
```

---

## `lea`

```asm
lea rax, [rbp-0x10]
```

Often used to calculate an address.

Think:

```c
rax = &variable;
```

But `lea` does not necessarily mean "load a pointer"; it can also perform arithmetic.

---

## `add`

```asm
add eax, 1
```

Equivalent to:

```c
eax++;
```

---

## `sub`

```asm
sub eax, 5
```

Equivalent to:

```c
eax -= 5;
```

---

## `xor`

```asm
xor eax, eax
```

Common way to set a register to zero:

```c
eax = 0;
```

---

## `cmp`

```asm
cmp eax, ebx
```

Conceptually:

```c
compare(eax, ebx);
```

Usually followed by a conditional jump.

---

# 15. Conditional Jumps

These are extremely important.

```text
je / jz    equal / zero
jne / jnz  not equal / not zero
jg         greater
jge        greater or equal
jl         less
jle        less or equal
ja         unsigned above
jb         unsigned below
```

Example:

```asm
cmp eax, 0x1337
jne fail
```

Equivalent to:

```c
if (eax != 0x1337)
    goto fail;
```

Another:

```asm
cmp eax, ebx
je success
```

Equivalent to:

```c
if (eax == ebx)
    goto success;
```

---

# 16. Recognizing Loops

Assembly:

```asm
add DWORD PTR [rbp-0x4], 1
cmp DWORD PTR [rbp-0x4], 0x1c
jbe loop
```

Likely means:

```c
i++;

if (i <= 28)
    goto loop;
```

Often reconstruct it as:

```c
for (int i = 0; i <= 28; i++) {
    ...
}
```

---

# 17. Recognizing `if` Statements

Assembly:

```asm
cmp eax, 0
jne fail

; success code
```

Means approximately:

```c
if (eax != 0)
    fail();
```

Or:

```c
if (eax == 0) {
    success();
}
```

Always identify which branch is success and which is failure.

---

# 18. Recognizing Function Calls

Assembly:

```asm
mov edi, eax
call rand
```

Look for:

```asm
call function
```

Then determine what arguments are being placed into:

```text
RDI
RSI
RDX
RCX
R8
R9
```

before the call.

The return value is generally in:

```text
RAX
```

---

# 19. Common Interesting Functions

When you see these, investigate them.

### Input

```text
scanf
fgets
gets
read
recv
argv
```

### Comparison

```text
strcmp
strncmp
memcmp
```

### Randomness

```text
rand
srand
random
srandom
```

### Crypto

```text
AES
EVP_*
SHA*
MD5
HMAC
```

### Memory

```text
memcpy
memmove
memset
malloc
free
```

### File operations

```text
open
read
fopen
fread
```

---

# 20. `strcmp()` Pattern

Suppose you see:

```asm
call strcmp
test eax, eax
jne fail
```

`strcmp()` returns:

```text
0 → strings equal
non-zero → strings different
```

So this likely means:

```c
if (strcmp(input, expected) != 0)
    fail();
```

This is often a very easy way to find a hardcoded password.

---

# 21. `memcmp()` Pattern

Example:

```asm
call memcmp
test eax, eax
jne fail
```

Likely:

```c
if (memcmp(input, expected, length) != 0)
    fail();
```

Look for the arguments before `memcmp()`.

---

# 22. `strlen()` Pattern

If you see:

```asm
call strlen
cmp eax, 0x20
jne fail
```

Likely:

```c
if (strlen(input) != 32)
    fail();
```

This tells you the expected input length.

---

# 23. Random Number Challenges

If you see:

```text
srand()
rand()
```

stop and investigate.

Remember:

```c
srand(seed);
rand();
```

is deterministic.

If the seed is predictable or has a small search space:

```text
brute force it
```

Example:

```python
for seed in range(256):
    random.seed(seed)
    ...
```

**Important:** Python's `random` is NOT the same implementation as C's `rand()`.

For a Linux binary using glibc, reproduce the actual libc behavior, usually with C.

Example:

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    for (int seed = 0; seed < 256; seed++) {
        srand(seed);
        printf("%d -> %d\n", seed, rand());
    }
}
```

---

# 24. Hexadecimal Basics

You will constantly encounter hex.

Useful conversions:

```text
0x00 = 0
0x01 = 1
0x0a = 10
0x10 = 16
0x1c = 28
0x20 = 32
0x41 = 'A'
0x42 = 'B'
0x61 = 'a'
0x62 = 'b'
0x7b = '{'
0x7d = '}'
```

ASCII:

```bash
man ascii
```

Or:

```python
chr(0x41)
```

---

# 25. Decimal ↔ Hex

Linux:

```bash
printf '%x\n' 1234
```

Python:

```python
hex(1234)
```

Reverse:

```python
int("4d2", 16)
```

---

# 26. ASCII Tricks

If you see numbers such as:

```text
72
84
66
123
```

try ASCII:

```text
72  → H
84  → T
66  → B
123 → {
```

You can use:

```python
print(''.join(map(chr, [72,84,66,123])))
```

Output:

```text
HTB{
```

---

# 27. Finding Hidden Arrays

If `nm` shows:

```text
0000000000004080 D check
```

inspect it:

```gdb
x/20wx &check
```

Or:

```gdb
x/20dw &check
```

If you see repeated values, that can be a clue.

For example:

```text
1127757600
1127757600
1127757600
```

could indicate repeated characters or repeated states.

---

# 28. PIE Address Confusion

You may see:

```text
main = 0x1185
```

but while running GDB:

```text
main = 0x555555555185
```

That's normal for PIE.

The binary's:

```text
0x1185
```

is an offset.

At runtime it gets a base address such as:

```text
0x555555554000
```

and:

```text
0x555555554000 + 0x1185
=
0x555555555185
```

Using symbols in GDB usually handles this for you.

---

# 29. Useful GDB Commands

## Navigation

```gdb
start
run
continue
```

## Breakpoints

```gdb
b main
b *0x1234
info breakpoints
delete 1
```

## Stepping

```gdb
si
ni
```

## Registers

```gdb
info registers
p/x $rax
p/x $rdi
p/x $rip
```

## Memory

```gdb
x/10x $rsp
x/20i $rip
x/s $rdi
x/20wx ADDRESS
```

## Code

```gdb
disassemble main
disassemble /m main
```

## Information

```gdb
info functions
info files
info registers
```

---

# 30. A Very Useful GDB Workflow

When you find an interesting function:

```gdb
gdb -q ./chall
```

Then:

```gdb
set disassembly-flavor intel
set pagination off
start
```

Find the function:

```gdb
info functions
```

Disassemble:

```gdb
disassemble main
```

Set a breakpoint:

```gdb
b *main+100
```

Run:

```gdb
run
```

Inspect:

```gdb
info registers
```

Step:

```gdb
ni
```

Repeat while watching:

```text
input
↓
calculation
↓
comparison
↓
branch
```

---

# 31. Dynamic Analysis Strategy

When static analysis becomes confusing:

```text
Don't stare at the assembly forever.
Run the program.
```

Set a breakpoint immediately before an important operation.

For example:

```asm
call strcmp
```

Break before it:

```gdb
b *ADDRESS
```

Then inspect:

```gdb
x/s $rdi
x/s $rsi
```

You might discover:

```text
$ rdi → user input
$ rsi → expected password
```

Instantly revealing the comparison.

---

# 32. Static vs Dynamic Analysis

## Static Analysis

You don't run the program.

Tools:

```text
strings
nm
readelf
objdump
Ghidra
IDA
Binary Ninja
```

Useful for:

```text
program structure
functions
constants
strings
algorithms
control flow
```

---

## Dynamic Analysis

You run the program.

Tools:

```text
gdb
gdb-peda
pwndbg
GEF
strace
ltrace
```

Useful for:

```text
runtime values
registers
memory
function arguments
branches
program state
```

### Good RE habit

Use both.

```text
STATIC
  ↓
find interesting code
  ↓
DYNAMIC
  ↓
observe actual values
  ↓
STATIC
  ↓
understand algorithm
```

---

# 33. `ltrace`

Useful for dynamically observing library calls:

```bash
ltrace ./chall
```

Especially useful for:

```text
strcmp
strlen
scanf
printf
rand
srand
malloc
```

Example:

```text
strcmp("hello", "password") = -8
```

This can immediately reveal what the program is comparing.

---

# 34. `strace`

System-call tracing:

```bash
strace ./chall
```

Useful for:

```text
open
read
write
execve
connect
access
```

This is particularly useful if the binary reads:

```text
files
configuration
environment variables
```

or communicates over the network.

---

# 35. Ghidra

For larger binaries, use Ghidra.

Basic workflow:

```text
Import binary
    ↓
Analyze
    ↓
Functions
    ↓
main()
    ↓
Decompiler
    ↓
Find input
    ↓
Find validation
```

The decompiler may turn:

```asm
cmp eax, 0x1337
jne fail
```

into something much easier:

```c
if (value != 0x1337) {
    fail();
}
```

This is extremely useful when the assembly becomes complicated.

---

# 36. What to Search For in Ghidra

Use:

```text
Search → For Strings
```

Search:

```text
flag
correct
wrong
success
password
key
secret
```

Then:

```text
Right click string
→ References
```

This can take you directly to the function using that string.

---

# 37. Common Beginner RE Patterns

## Pattern 1 — Hardcoded Password

```c
if (strcmp(input, "secret123") == 0)
    win();
```

Solution:

```text
secret123
```

---

## Pattern 2 — Numeric Comparison

```c
if (input == 1337)
    win();
```

Look for:

```asm
cmp eax, 0x539
```

because:

```text
0x539 = 1337
```

---

## Pattern 3 — Character-by-Character Validation

```c
if (input[0] != 'H') fail();
if (input[1] != 'T') fail();
if (input[2] != 'B') fail();
```

Look for repeated:

```asm
cmp BYTE PTR [...]
```

---

## Pattern 4 — XOR

Example:

```c
for (i = 0; i < n; i++)
    output[i] = input[i] ^ 0x42;
```

XOR is reversible:

```text
A XOR B XOR B = A
```

So encryption/decryption often uses the same operation.

---

## Pattern 5 — Addition/Subtraction

```c
encoded[i] = input[i] + 5;
```

Reverse:

```c
input[i] = encoded[i] - 5;
```

---

## Pattern 6 — ROT / Caesar

Example:

```text
A → D
B → E
C → F
```

Try:

```text
Caesar cipher
ROT13
```

---

## Pattern 7 — Hash Comparison

If you see:

```text
MD5
SHA1
SHA256
```

the program may hash your input and compare it against a hardcoded hash.

Then determine:

```text
algorithm
input format
salt
iterations
comparison
```

---

# 38. Recognizing XOR in Assembly

Common pattern:

```asm
xor eax, DWORD PTR [...]
```

or:

```asm
xor BYTE PTR [...], 0x42
```

Think:

```c
value ^= key;
```

Repeated XOR with the same key can often be reversed easily.

---

# 39. Recognizing Bit Operations

Common instructions:

```text
and
or
xor
shl
shr
rol
ror
```

Examples:

```asm
shl eax, 1
```

Approximately:

```c
eax <<= 1;
```

```asm
shr eax, 2
```

Approximately:

```c
eax >>= 2;
```

---

# 40. Watch for Integer Tricks

CTFs love:

```text
overflow
underflow
signed vs unsigned
integer truncation
byte/word/dword conversion
```

Pay attention to:

```asm
movzx
movsx
```

### `movzx`

Move and zero extend.

Example:

```asm
movzx eax, byte ptr [...]
```

A byte:

```text
0xff
```

becomes:

```text
0x000000ff
```

---

### `movsx`

Move and sign extend.

For example:

```text
0xff
```

as a signed byte is:

```text
-1
```

This distinction can matter enormously.

---

# 41. Signed vs Unsigned Comparisons

These are different:

```text
jg / jl
```

versus:

```text
ja / jb
```

Generally:

```text
jg/jl → signed
ja/jb → unsigned
```

If a comparison doesn't make sense mathematically, check whether the program is treating the value as signed or unsigned.

---

# 42. When You Get Stuck

Don't immediately start randomly trying things.

Ask:

```text
1. Where does input enter?
2. Where is it stored?
3. What operations happen to it?
4. What is it compared against?
5. Where is the failure branch?
6. Where is the success branch?
7. Can I reproduce the calculation?
8. Is there a small search space?
```

---

# 43. Search Space Thinking

Always estimate the search space.

Examples:

```text
1 byte          → 256
2 bytes         → 65,536
4 decimal digits → 10,000
6 decimal digits → 1,000,000
printable ASCII → ~95^N
```

If it's small:

```text
BRUTE FORCE
```

If it's huge:

```text
LOOK FOR STRUCTURE
```

For example, in FlagCasino:

```text
input = 1 byte
```

Therefore:

```text
256 possibilities
```

That's trivial.

---

# 44. Don't Confuse Obfuscation With Encryption

If you see something complicated like:

```c
x = ((input ^ 0x42) + 17) * 3;
```

don't assume it's cryptography.

Often it's just:

```text
obfuscation
```

Reverse the operations in the opposite order:

```text
x / 3
x - 17
x XOR 0x42
```

---

# 45. Common Tools

## Basic Linux

```text
file
strings
grep
xxd
hexdump
objdump
readelf
nm
ldd
```

## Debugging

```text
gdb
pwndbg
GEF
gdb-peda
```

## Reverse Engineering

```text
Ghidra
IDA
Binary Ninja
Cutter
radare2
```

## Tracing

```text
strace
ltrace
```

---

# 46. My Recommended Beginner Toolset

You don't need everything.

Start with:

```text
file
strings
nm
readelf
objdump
gdb
Ghidra
```

Then eventually add:

```text
pwndbg
ltrace
strace
radare2
```

---

# 47. A Practical RE Workflow

When given:

```text
chall
```

do this:

### Step 1

```bash
file chall
```

### Step 2

```bash
checksec --file=chall
```

### Step 3

```bash
strings -a -n 4 chall | less
```

### Step 4

```bash
nm -C chall | less
```

### Step 5

```bash
./chall
```

### Step 6

```bash
gdb -q ./chall
```

### Step 7

```gdb
set disassembly-flavor intel
set pagination off
info functions
```

### Step 8

```gdb
disassemble main
```

### Step 9

Identify:

```text
INPUT
VALIDATION
SUCCESS
FAILURE
```

### Step 10

Inspect interesting memory:

```gdb
x/20wx ADDRESS
x/s ADDRESS
```

### Step 11

Break at interesting operations:

```gdb
b *ADDRESS
run
```

### Step 12

Inspect registers:

```gdb
info registers
```

### Step 13

Step through:

```gdb
ni
```

### Step 14

Reconstruct pseudocode.

### Step 15

Write a solver.

---

# 48. The RE Mindset

Don't ask:

> "How do I understand this entire binary?"

Ask:

> "What does the program need me to provide?"

Then:

> "How does it determine whether I'm correct?"

Then:

> "Can I reproduce or reverse that calculation?"

This dramatically reduces the amount of code you need to understand.

---

# 49. FlagCasino Example

The entire challenge can be reduced to:

```text
                casino
                   │
                   ▼
             read 1 byte
                   │
                   ▼
             srand(input)
                   │
                   ▼
               rand()
                   │
                   ▼
             compare with
               check[i]
                   │
             ┌─────┴─────┐
             │           │
          equal       different
             │           │
             ▼           ▼
          continue      FAIL
             │
             ▼
          repeat
             │
             ▼
         29 characters
             │
             ▼
 HTB{r4nd_1s_v3ry_pr3d1ct4bl3}
```

The challenge looked like a casino.

The actual vulnerability/weakness was simply:

```text
deterministic PRNG + tiny seed space
```

---

# 50. Quick Reference Card

```text
┌──────────────────────────────────────────────┐
│             RE CTF QUICK REFERENCE           │
├──────────────────────────────────────────────┤
│ file ./chall                                 │
│ checksec --file=./chall                      │
│ strings -a -n 4 ./chall                      │
│ nm -C ./chall                                │
│ readelf -h ./chall                           │
│ readelf -S ./chall                           │
│ objdump -d -M intel ./chall                  │
│                                              │
│ gdb -q ./chall                               │
│ set disassembly-flavor intel                 │
│ set pagination off                            │
│ info functions                               │
│ start                                        │
│ disassemble main                             │
│ b main                                       │
│ b *ADDRESS                                   │
│ run                                          │
│ c                                            │
│ si                                           │
│ ni                                           │
│ info registers                               │
│ p/x $rax                                     │
│ x/20wx ADDRESS                               │
│ x/s ADDRESS                                  │
│ x/20i $rip                                   │
│                                              │
│ INPUT → PROCESS → COMPARE → SUCCESS/FAIL     │
└──────────────────────────────────────────────┘
```

---

# 51. Final Checklist

Before declaring yourself stuck, check:

```text
[ ] Did I run the binary?
[ ] Did I run strings?
[ ] Did I check symbols?
[ ] Did I find main()?
[ ] Did I find where input enters?
[ ] Did I find the validation logic?
[ ] Did I identify the success branch?
[ ] Did I inspect constants/arrays?
[ ] Did I check for strcmp/memcmp?
[ ] Did I check for rand/srand?
[ ] Did I check for XOR?
[ ] Did I check for arithmetic transformations?
[ ] Did I check ASCII values?
[ ] Did I consider signed/unsigned conversion?
[ ] Did I estimate the brute-force search space?
[ ] Did I reproduce the algorithm independently?
[ ] Did I use GDB to verify my theory?
```

---

# 52. Golden Rule

The most important rule for beginner RE CTFs:

> **You don't need to understand everything. You need to understand enough.**

Find:

```text
INPUT
  ↓
WHAT HAPPENS TO IT?
  ↓
WHAT IS IT COMPARED AGAINST?
  ↓
WHAT MAKES THE PROGRAM SAY "CORRECT"?
```

Once you can answer those four questions, you can solve a surprisingly large number of beginner reverse-engineering challenges.
