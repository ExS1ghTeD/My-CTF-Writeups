# HTB — Satellite Hijack

**Category:** Reverse Engineering
**Difficulty:** Hard
**Platform:** Hack The Box
**Files:** `satellite`, `library.so`

---

## 1. Challenge Description

We found an old control panel that was previously used to communicate with an observation satellite.

The system appears to have been tampered with, preventing the original control codes from being transmitted.

Our goal is to reverse engineer the software, recover the hidden control code, and ultimately obtain the flag.

The challenge provides two files:

```text
satellite
library.so
```

At first glance, this looks like a normal ELF reverse-engineering challenge. However, the interesting logic is hidden inside the shared library.

---

# 2. Initial Reconnaissance

Whenever I start a reverse-engineering challenge, I first want to answer a few basic questions:

1. What type of files are these?
2. Are they stripped?
3. Is the binary dynamically linked?
4. What protections are enabled?
5. What functions and strings are visible?

Start with:

```bash
file satellite
file library.so
```

Output:

```text
satellite: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
for GNU/Linux 3.2.0, not stripped

library.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV),
dynamically linked, stripped
```

This already gives us an important clue.

`satellite` is **not stripped**, while `library.so` **is stripped**.

That means the executable should be easier to inspect, while the shared library will require more manual reverse engineering.

---

# 3. Check Binary Protections

Next:

```bash
checksec --file=./satellite
```

We get:

```text
RELRO           Partial RELRO
STACK CANARY    No canary found
NX              NX enabled
PIE             PIE enabled
RPATH           No RPATH
RUNPATH         No RUNPATH
Symbols         71
FORTIFY         No
```

The interesting part for this challenge is:

```text
Partial RELRO
```

Why?

ELF programs use something called the **Global Offset Table (GOT)** to store addresses of dynamically imported functions.

For example, the program imports:

```text
read()
printf()
puts()
```

The GOT contains the runtime addresses of those functions.

With **Full RELRO**, the GOT becomes read-only after relocation.

With **Partial RELRO**, some GOT entries can remain writable.

That means GOT overwriting/hijacking may be possible.

We should keep that clue in mind.

---

# 4. Running the Program

Run it normally:

```bash
./satellite
```

We get:

```text
         ,-.
        / \  `.  __..-,O ≈ ≈ ≈ ≈ ≈ ≈ ≈ ≈ ≈ ≈ ≈ ≈
       :   \ --''_..-'.'
       |    . .-' `. '.
       :     .     .`.'
        \     `.  /  ..
        \      `.   ' .
          `,       `.   \
         ,|,`.        `-.\
         '.||  ``-...__..-`
          |  |
          |__|
          /||\
         //||\\
        // || \\
    __//__||__\\__
   '--------------'
| READY TO TRANSMIT |
>
```

It accepts input:

```text
> help
Sending `help`

> test
Sending `test`

> HTB
Sending `HTB`
```

Nothing immediately suspicious happens.

This suggests the visible program may just be a frontend.

---

# 5. Inspecting Strings

Let's look at the strings:

```bash
strings -a -n 4 satellite
```

One particularly interesting section is:

```text
send_satellite_message
./library.so
```

This is a huge clue.

The executable imports a function called:

```text
send_satellite_message
```

but that function isn't implemented inside the executable.

We can confirm this using:

```bash
nm -C satellite
```

We see:

```text
                 U send_satellite_message
```

The `U` means **undefined**.

In other words:

> `satellite` expects another library to provide `send_satellite_message()`.

And we already have:

```text
library.so
```

So our basic program structure becomes:

```text
satellite
    |
    | calls
    v
send_satellite_message()
    |
    | implemented by
    v
library.so
```

This tells us where to focus next.

---

# 6. Looking at `main()`

We can inspect the functions:

```bash
gdb -q ./satellite
```

Then:

```gdb
info functions
```

We see:

```text
0x0000000000001185  main
0x0000000000001080  send_satellite_message@plt
```

The interesting function is obviously:

```text
main
```

Disassemble it:

```bash
objdump -d -M intel satellite
```

or inside GDB:

```gdb
disassemble main
```

Eventually we see a call through the PLT:

```asm
call send_satellite_message@plt
```

So the executable itself isn't doing anything particularly complicated.

The real logic is in:

```text
library.so
```

---

# 7. Inspecting the Shared Library

Let's start with the usual commands:

```bash
readelf -h library.so
readelf -S library.so
readelf -d library.so
readelf -sW library.so
```

Also:

```bash
nm -D library.so
```

and:

```bash
objdump -T library.so
```

The library is stripped, so we don't get nice function names.

That's normal in CTFs.

When symbols are stripped, we can instead use:

* strings
* imported functions
* assembly
* cross-references
* GDB
* ELF structures
* dynamic linking information

---

# 8. Searching for Interesting Strings

Run:

```bash
strings -a -tx library.so
```

The `-tx` option is useful because it also prints the offset where each string was found.

At first, many things look strange.

This is a sign that the library contains some form of obfuscation.

One important technique used in this challenge is related to:

```text
memfrob()
```

`memfrob()` is a very simple obfuscation function.

Conceptually, it does:

```c
for (int i = 0; i < length; i++)
    buffer[i] ^= 0x2a;
```

So every byte is XORed with:

```text
0x2a
```

Because XOR is reversible, doing the same operation again decrypts it:

```text
encrypted ^ 0x2a = plaintext
```

and:

```text
plaintext ^ 0x2a = encrypted
```

This is an important beginner lesson:

> XOR with the same key twice gives you the original value.

---

# 9. Finding the Hidden Environment Variable

One of the obfuscated strings can be decoded by reversing the transformation.

The bytes eventually reveal:

```text
SAT_PROD_ENVIRONRONMENT
```

Notice the unusual spelling:

```text
ENVIRONRONMENT
```

It is tempting to assume this is a typo.

Don't.

In reverse-engineering challenges, unusual strings are often deliberate clues.

Set the variable:

```bash
export SAT_PROD_ENVIRONRONMENT=1
```

Then run:

```bash
./satellite
```

This activates the hidden functionality.

---

# 10. Why Does the Environment Variable Matter?

The library contains malicious/backdoor functionality that is only enabled when the special environment variable exists.

Conceptually:

```c
if (getenv("SAT_PROD_ENVIRONRONMENT")) {
    // activate hidden functionality
}
```

The real implementation is more complicated, but this is the basic idea.

This is a useful reverse-engineering technique to remember:

> If a binary behaves differently under certain environment variables, search for `getenv()` and investigate the strings passed to it.

---

# 11. Understanding the GOT

The next important concept is the **Global Offset Table**, or GOT.

Suppose the program calls:

```c
read(0, buffer, size);
```

The actual address of `read()` isn't necessarily known when the program is compiled.

Instead, dynamic linking resolves it at runtime.

The program eventually calls something similar to:

```text
read@plt
    |
    v
read@got
    |
    v
libc read()
```

The important part is:

```text
GOT entry → actual function address
```

If an attacker can modify the GOT entry:

```text
read@got
```

they can redirect the program somewhere else.

For example:

```text
Before:

read@got → libc read()


After:

read@got → attacker_function()
```

This is called **GOT hijacking**.

And remember our earlier observation:

```text
Partial RELRO
```

That makes this technique possible.

---

# 12. What the Library Does

The malicious library dynamically examines the program's ELF structures.

It uses information such as:

```text
Program headers
Dynamic section
Symbol table
String table
Relocations
```

to locate imported functions.

Eventually it finds the GOT entry associated with:

```text
read()
```

and changes it.

Conceptually:

```text
library.so
    |
    v
find program headers
    |
    v
find dynamic section
    |
    v
find relocation information
    |
    v
find read@GOT
    |
    v
overwrite read@GOT
```

After that, calls to `read()` can be redirected to the hidden code.

This is a great example of why understanding ELF internals is useful in reverse engineering.

---

# 13. Deobfuscating the Library

The interesting obfuscated region begins around:

```text
0x11a9
```

The challenge uses XOR `0x2a` as an obfuscation layer.

We can make a working copy:

```bash
cp library.so library.original.so
```

Then create a small Python script:

```python
from pathlib import Path

data = bytearray(Path("library.so").read_bytes())

start = 0x11a9

for i in range(start, len(data)):
    data[i] ^= 0x2a

Path("library_deobf.so").write_bytes(data)
```

Save it as:

```text
decrypt.py
```

Then:

```bash
python3 decrypt.py
```

Now inspect the result:

```bash
file library_deobf.so
```

and:

```bash
objdump -d -M intel library_deobf.so
```

The assembly around the interesting area is now much easier to understand.

---

# 14. Finding the Embedded Key

While examining the deobfuscated assembly, we encounter several `movabs` instructions.

These load large constants into registers.

For example:

```text
0x3759305630307b356c
0x3a7c3e753f665666
0x784c7c214f3a7c3e
0x00663b2c6a24216f
```

At first these look like random numbers.

But this is where we need to remember something important about x86-64:

> x86 is little-endian.

For example, if memory contains:

```text
6c 35 7b 30 76 30 59 37
```

the 64-bit integer representation looks reversed when displayed as a number.

So we need to convert these constants back into bytes.

The four chunks correspond to:

```text
l5{0v0Y7
fVf?u>|:
>|:O!|Lx
!o$j,;f
```

---

# 15. The Overlapping Writes

This is one of the trickiest parts of the challenge.

We might initially think the four strings are simply concatenated:

```text
chunk1 + chunk2 + chunk3 + chunk4
```

But that's not what the assembly does.

The writes occur at different offsets.

Conceptually:

```text
chunk 1 → offset 0
chunk 2 → offset 8
chunk 3 → offset 13
chunk 4 → offset 21
```

Notice something:

```text
chunk 2
offset 8
length 8

therefore:

8 9 10 11 12 13 14 15
```

But chunk 3 begins at:

```text
13
```

So chunk 3 overwrites part of chunk 2.

Visualizing it:

```text
Offset:

0       8       13      21
|-------|-------|-------|
        chunk 2
                chunk 3
                        chunk 4
```

The important lesson is:

> Don't assume multiple immediate writes are independent strings. Always pay attention to their destination addresses.

This is especially important when reading compiler-generated assembly.

---

# 16. Reconstructing the Key

We can reproduce the writes in Python:

```python
s1 = b"l5{0v0Y7"
s2 = b"fVf?u>|:"
s3 = b">|:O!|Lx"
s4 = b"!o$j,;f\x00"

key = bytearray(29)

key[0:8] = s1
key[8:16] = s2
key[13:21] = s3
key[21:29] = s4

print(key)
```

This gives us the effective key:

```text
l5{0v0Y7fVf?u>|:O!|Lx!o$j,;f
```

The important word here is **effective**.

This is the actual byte sequence after all the overlapping writes have happened.

---

# 17. Understanding the Final Verification

Now we inspect the code that validates the input.

The important operation is essentially:

```c
if ((input[i] ^ key[i]) == i)
```

Let's slow down and understand this.

Suppose:

```text
input[i] XOR key[i] = i
```

We want to find:

```text
input[i]
```

XOR has a useful property:

```text
A XOR B = C

therefore:

A = B XOR C
```

So:

```text
input[i] = key[i] XOR i
```

That's all we need.

---

# 18. Recovering the Flag

Use:

```python
s1 = b"l5{0v0Y7"
s2 = b"fVf?u>|:"
s3 = b">|:O!|Lx"
s4 = b"!o$j,;f\x00"

key = bytearray(29)

key[0:8] = s1
key[8:16] = s2
key[13:21] = s3
key[21:29] = s4

flag = bytes(key[i] ^ i for i in range(27))

print(flag.decode())
```

Running it gives:

```text
l4y3r5_0n_l4y3r5_0n_l4y3r5!}
```

Therefore the complete flag is:

```text
HTB{l4y3r5_0n_l4y3r5_0n_l4y3r5!}
```

---

# 19. Complete Attack Chain

The challenge becomes much easier to understand when we look at the entire process:

```text
                    satellite
                       |
                       |
              imports library.so
                       |
                       v
                library.so
                       |
                       v
       hidden environment variable
                       |
                       v
       SAT_PROD_ENVIRONRONMENT
                       |
                       v
              activate backdoor
                       |
                       v
          inspect ELF structures
                       |
                       v
                 find read@GOT
                       |
                       v
                hijack read()
                       |
                       v
              hidden verification
                       |
                       v
             XOR 0x2a obfuscation
                       |
                       v
             deobfuscate assembly
                       |
                       v
             recover 4 constants
                       |
                       v
             account for overlapping
                    writes
                       |
                       v
                  recover key
                       |
                       v
          input[i] XOR key[i] == i
                       |
                       v
             input[i] = key[i] XOR i
                       |
                       v
                    FLAG
```

---

# 20. Final Flag

```text
HTB{l4y3r5_0n_l4y3r5_0n_l4y3r5!}
```

---

# 21. What I Learned From This Challenge

This challenge was difficult because it combines several different reverse-engineering concepts.

The most important things I would take away are:

## 21.1 Always inspect imported functions

If you see:

```text
U some_function
```

in `nm`, it means the function isn't implemented in that binary.

Look at the libraries.

For example:

```text
U send_satellite_message
```

immediately led us to:

```text
library.so
```

---

## 21.2 Stripped does not mean impossible

A stripped binary simply removes convenient symbol names.

You can still analyze:

```text
strings
imports
exports
assembly
ELF headers
GOT
PLT
relocations
runtime behavior
```

---

## 21.3 Partial RELRO should make you think about the GOT

When you see:

```text
Partial RELRO
```

ask:

> Can the challenge be modifying a GOT entry?

Especially if the program dynamically imports functions such as:

```text
read
write
printf
puts
```

---

## 21.4 Learn XOR

XOR appears everywhere in CTFs.

Important identities:

```text
A ^ 0 = A

A ^ A = 0

A ^ B = C
B ^ C = A
A ^ C = B
```

The last property makes XOR useful for both encryption and obfuscation.

---

## 21.5 Pay attention to endianness

When you see something like:

```asm
movabs rax, 0x3759305630307b356c
```

don't immediately treat it as a number.

Ask:

> Could this be ASCII stored as a 64-bit integer?

On x86-64, remember:

```text
little endian
```

So the bytes are stored least-significant byte first.

---

## 21.6 Watch for overlapping writes

If assembly does:

```asm
mov [buffer], ...
mov [buffer+8], ...
mov [buffer+13], ...
```

don't automatically concatenate everything.

Draw the offsets:

```text
0
        8
             13
                     21
```

Then determine which bytes overwrite previous bytes.

---

## 21.7 Don't trust strings blindly

This challenge deliberately contains weird-looking data.

For example:

```text
SAT_PROD_ENVIRONRONMENT
```

looks like a typo.

But in a CTF, strange strings are often intentional.

Instead of "fixing" the string mentally, use exactly what the binary uses.

---

# 22. Useful Commands for Similar Challenges

Here's the small toolkit I would keep for future ELF reverse-engineering challenges.

### Identify the file

```bash
file ./binary
```

### Check protections

```bash
checksec --file=./binary
```

### Find strings

```bash
strings -a ./binary
```

### Find strings with offsets

```bash
strings -a -tx ./binary
```

### List symbols

```bash
nm -C ./binary
```

### List dynamic symbols

```bash
nm -D ./binary
```

### Inspect ELF headers

```bash
readelf -h ./binary
```

### Inspect sections

```bash
readelf -S ./binary
```

### Inspect dynamic information

```bash
readelf -d ./binary
```

### Inspect symbols

```bash
readelf -sW ./binary
```

### Inspect relocations

```bash
readelf -rW ./binary
```

### Inspect assembly

```bash
objdump -d -M intel ./binary
```

### Start GDB

```bash
gdb -q ./binary
```

### Useful GDB commands

```gdb
info functions
info registers
info proc mappings
disassemble main
x/20gx ADDRESS
x/s ADDRESS
break main
run
continue
```

---

# 23. Beginner Mental Model

When you get another reverse-engineering CTF, don't immediately try to understand every instruction.

Start with questions:

```text
What is this file?
        ↓
What protections does it have?
        ↓
Is it stripped?
        ↓
What functions does it import?
        ↓
What interesting strings exist?
        ↓
Where does user input enter?
        ↓
Where is that input processed?
        ↓
Is there an obvious comparison?
        ↓
Is there encryption/encoding/obfuscation?
        ↓
Can I reverse that transformation?
```

For this challenge, that thought process eventually became:

```text
satellite
   ↓
library.so
   ↓
hidden environment variable
   ↓
GOT manipulation
   ↓
read() hijacking
   ↓
XOR obfuscation
   ↓
little-endian constants
   ↓
overlapping writes
   ↓
XOR verification
   ↓
flag
```

That's the real lesson of the challenge.

You don't need to understand the entire binary at once.

**Find one interesting clue, follow it, and let it lead you to the next one.**
