# FlagCasino — Reverse Engineering Writeup

## Challenge Information

| Field        | Value                           |
| ------------ | ------------------------------- |
| Challenge    | FlagCasino                      |
| Category     | Reverse Engineering             |
| Binary       | `casino`                        |
| Architecture | ELF 64-bit x86-64               |
| Difficulty   | Beginner                        |
| Flag         | `HTB{r4nd_1s_v3ry_pr3d1ct4bl3}` |

---

## 1. Challenge Description

The challenge presents an abandoned casino where a robotic dealer challenges us to beat the house.

We are given an executable named `casino`.

The goal is to reverse engineer the binary and recover the flag.

---

# 2. Initial Reconnaissance

First, identify the binary:

```bash
file casino
```

Output:

```text
casino: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
BuildID[sha1]=ac3d9d8a2c65ca7a0cb88af07efaec8c991c315d,
for GNU/Linux 3.2.0, not stripped
```

The important detail here is:

```text
not stripped
```

This means useful symbol information is still present in the binary.

We can therefore use tools such as `nm` and GDB to find functions.

---

# 3. Looking for Interesting Strings

We can search the binary for obvious flag-related strings:

```bash
strings -a -n 4 casino | grep -iE 'flag|win|jackpot|money|cash|casino|congrat|prize'
```

The only interesting result is:

```text
[ ** WELCOME TO ROBO CASINO **]
```

There is no plaintext flag in the binary.

Therefore, we need to inspect the program logic.

---

# 4. Looking at the Functions

Start GDB:

```bash
gdb -q ./casino
```

Then:

```gdb
set disassembly-flavor intel
set pagination off
info functions
```

The important functions are:

```text
0x0000000000001185  main
0x0000000000001030  puts@plt
0x0000000000001040  printf@plt
0x0000000000001050  srand@plt
0x0000000000001060  __isoc99_scanf@plt
0x0000000000001070  exit@plt
0x0000000000001080  rand@plt
```

The presence of both:

```text
srand()
rand()
```

is immediately interesting because these functions are used to generate deterministic pseudo-random numbers.

---

# 5. Disassembling `main()`

Run:

```gdb
disassemble main
```

The important portion is:

```asm
0x11ce <+73>:    lea    rax,[rbp-0x5]
0x11d2 <+77>:    mov    rsi,rax
0x11d5 <+80>:    lea    rdi,[rip+0xf20]
0x11dc <+87>:    mov    eax,0x0
0x11e1 <+92>:    call   0x1060 <__isoc99_scanf@plt>
```

This shows that the program reads user input using `scanf()`.

Immediately afterwards:

```asm
0x11f5 <+112>:   movzx  eax,BYTE PTR [rbp-0x5]
0x11f9 <+116>:   movsx  eax,al
0x11fc <+119>:   mov    edi,eax
0x11fe <+121>:   call   0x1050 <srand@plt>
```

The input is passed to:

```c
srand(input);
```

Notice that the input is loaded as a **byte**.

This means there are only 256 possible byte values to consider.

---

# 6. Finding the Random Number Check

Immediately after `srand()`:

```asm
0x1203 <+126>:   call   0x1080 <rand@plt>
```

So the program generates a pseudo-random number.

The result is then compared against an array called `check`.

The relevant instructions are:

```asm
0x1216 <+145>:   lea    rdx,[rip+0x2e63]        # 0x4080 <check>
0x121d <+152>:   mov    edx,DWORD PTR [rcx+rdx*1]
0x1220 <+155>:   cmp    eax,edx
0x1222 <+157>:   jne    0x1232
```

Conceptually, this is:

```c
random_number = rand();

if (random_number != check[i]) {
    // failure
}
```

The symbol table confirms that `check` is a global data object:

```bash
nm -C casino | grep check
```

Output:

```text
0000000000004080 D check
```

---

# 7. Understanding the Loop

At the beginning of `main()` we see:

```asm
0x11b1 <+44>:    mov    DWORD PTR [rbp-0x4],0x0
```

This initializes a counter:

```c
int i = 0;
```

At the end of the loop:

```asm
0x1254 <+207>:   add    DWORD PTR [rbp-0x4],0x1
```

So:

```c
i++;
```

The loop condition is:

```asm
0x1258 <+211>:   mov    eax,DWORD PTR [rbp-0x4]
0x125b <+214>:   cmp    eax,0x1c
0x125e <+217>:   jbe    0x11bd
```

`0x1c` is hexadecimal for decimal `28`.

Therefore the loop runs for:

```text
0 through 28
```

which is:

```text
29 iterations
```

The high-level logic is approximately:

```c
for (int i = 0; i <= 28; i++) {
    scanf(...);
    srand(input);
    value = rand();

    if (value != check[i]) {
        exit();
    }
}
```

---

# 8. Dumping the `check[]` Array

Since we know the address of `check` is `0x4080`, we can inspect it in GDB.

Start the program first so that GDB resolves the PIE address:

```gdb
start
```

Then:

```gdb
x/29wd &check
```

The output was:

```text
0x555555558080 <check>: 608905406       183990277       286129175       128959393
0x555555558090 <check+16>:      1795081523      1322670498      868603056       677741240
0x5555555580a0 <check+32>:      1127757600      89789692        421093279       1127757600
0x5555555580b0 <check+48>:      1662292864      1633333913      1795081523      1819267000
0x5555555580c0 <check+64>:      1127757600      255697463       1795081523      1633333913
0x5555555580d0 <check+80>:      677741240       89789692        988039572       114810857
0x5555555580e0 <check+96>:      1322670498      214780621       1473834340      1633333913
0x5555555580f0 <check+112>:     585743402
```

So the binary contains 29 predetermined random numbers.

---

# 9. The Key Observation: `rand()` Is Deterministic

The important property of C's `rand()` is that it is a **pseudo-random number generator**.

It is deterministic with respect to its seed.

For example:

```c
srand(1234);
printf("%d\n", rand());
```

will produce the same first value every time on the same libc implementation.

Therefore:

```text
srand(seed)
      ↓
    rand()
      ↓
 deterministic value
```

This means we can work backwards.

We know:

```text
rand() = check[i]
```

and we want:

```text
seed = ?
```

The seed is only one byte, so we can brute-force all 256 possible values.

---

# 10. Recovering the First Character

The first value is:

```text
check[0] = 608905406
```

Try:

```c
srand(72);
rand();
```

On the challenge's libc implementation:

```text
608905406
```

Therefore:

```text
seed = 72
```

ASCII decimal `72` is:

```text
H
```

So the first character is:

```text
H
```

---

# 11. Recovering All Characters

We can automate this.

Create `solve.c`:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    unsigned int check[] = {
        608905406, 183990277, 286129175, 128959393,
        1795081523, 1322670498, 868603056, 677741240,
        1127757600, 89789692, 421093279, 1127757600,
        1662292864, 1633333913, 1795081523, 1819267000,
        1127757600, 255697463, 1795081523, 1633333913,
        677741240, 89789692, 988039572, 114810857,
        1322670498, 214780621, 1473834340, 1633333913,
        585743402
    };

    for (int i = 0; i < 29; i++) {
        for (int c = 0; c < 256; c++) {
            srand((signed char)c);

            if ((unsigned int)rand() == check[i]) {
                putchar(c);
                break;
            }
        }
    }

    putchar('\n');

    return 0;
}
```

Compile:

```bash
gcc solve.c -o solve
```

Run:

```bash
./solve
```

Output:

```text
HTB{r4nd_1s_v3ry_pr3d1ct4bl3}
```

---

# 12. Recovered Characters

For reference, the mapping is:

| Index | `check[i]` | Seed | Character |
| ----: | ---------: | ---: | :-------: |
|     0 |  608905406 |   72 |    `H`    |
|     1 |  183990277 |   84 |    `T`    |
|     2 |  286129175 |   66 |    `B`    |
|     3 |  128959393 |  123 |    `{`    |
|     4 | 1795081523 |  114 |    `r`    |
|     5 | 1322670498 |   52 |    `4`    |
|     6 |  868603056 |  110 |    `n`    |
|     7 |  677741240 |  100 |    `d`    |
|     8 | 1127757600 |   95 |    `_`    |
|     9 |   89789692 |   49 |    `1`    |
|    10 |  421093279 |  115 |    `s`    |
|    11 | 1127757600 |   95 |    `_`    |
|    12 | 1662292864 |  118 |    `v`    |
|    13 | 1633333913 |   51 |    `3`    |
|    14 | 1795081523 |  114 |    `r`    |
|    15 | 1819267000 |  121 |    `y`    |
|    16 | 1127757600 |   95 |    `_`    |
|    17 |  255697463 |  112 |    `p`    |
|    18 | 1795081523 |  114 |    `r`    |
|    19 | 1633333913 |   51 |    `3`    |
|    20 |  677741240 |  100 |    `d`    |
|    21 |   89789692 |   49 |    `1`    |
|    22 |  988039572 |   99 |    `c`    |
|    23 |  114810857 |  116 |    `t`    |
|    24 | 1322670498 |   52 |    `4`    |
|    25 |  214780621 |   98 |    `b`    |
|    26 | 1473834340 |  108 |    `l`    |
|    27 | 1633333913 |   51 |    `3`    |
|    28 |  585743402 |  125 |    `}`    |

Combining the characters:

```text
HTB{r4nd_1s_v3ry_pr3d1ct4bl3}
```

---

# 13. Final Flag

```text
HTB{r4nd_1s_v3ry_pr3d1ct4bl3}
```

---

# 14. Lessons Learned

This challenge demonstrates several useful reverse-engineering concepts.

### 1. `not stripped` binaries are easier to analyze

When symbols are available, commands such as:

```bash
nm -C binary
```

and:

```gdb
info functions
```

can reveal useful function names.

---

### 2. Look for input functions

Interesting functions include:

```text
scanf
fgets
read
gets
```

These tell us where user-controlled data enters the program.

---

### 3. Look for comparisons

Assembly instructions such as:

```asm
cmp
test
je
jne
jz
jnz
```

often reveal the program's validation logic.

---

### 4. `rand()` + `srand()` should immediately attract attention

Pseudo-random number generators are deterministic.

If you can identify the seed or constrain the seed to a small range, you can reproduce the output.

---

### 5. Pay attention to data sizes

The program reads the seed as a byte:

```asm
movzx eax,BYTE PTR [...]
```

A byte gives us only:

```text
256 possible values
```

Brute-forcing 256 values is trivial.

---

### 6. Translate assembly into pseudocode

You don't need to understand every instruction immediately.

The important part is reconstructing something like:

```c
for (int i = 0; i < 29; i++) {
    input = get_input();

    srand(input);

    if (rand() != check[i]) {
        fail();
    }
}
```

Once the code is expressed at this level, the solution becomes much easier to see.

---

# 15. General RE Checklist

For future beginner reverse-engineering challenges, this is a useful checklist:

```text
[ ] file binary
[ ] check architecture
[ ] check whether stripped
[ ] run strings
[ ] inspect symbols with nm
[ ] inspect functions with GDB
[ ] find main()
[ ] disassemble main()
[ ] identify input
[ ] follow the input
[ ] identify comparisons
[ ] identify success/failure branches
[ ] inspect referenced data
[ ] determine whether values can be reproduced/brute-forced
[ ] write a small solver
```

The most important mindset is:

> **Don't try to understand the entire binary. Find the path from INPUT → VALIDATION → SUCCESS.**

That is usually enough to solve beginner and intermediate RE CTFs.
