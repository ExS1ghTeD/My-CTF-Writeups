# HTB — TunnelMadness

**Category:** Reverse Engineering
**Difficulty:** Medium
**Binary:** `tunnel`
**Remote:** `154.57.164.80:31062`

**Flag:**

```text
HTB{tunn3l1ng_ab0ut_in_3d_e5cef73b850211b7b9fc504adc4820d0}
```

---

## 1. Challenge Description

> Within Vault 8707 are located master keys used to access any vault in the country. Unfortunately, the entrance was caved in long ago. There are decades old rumors that the few survivors managed to tunnel out deep underground and make their way to safety. Can you uncover their tunnel and break back into the vault?

We are given an ELF binary called `tunnel`.

The program asks for movement directions:

```text
Direction (L/R/F/B/U/D/Q)?
```

The objective is to navigate through an underground tunnel and reach the vault.

---

# 2. Initial Binary Reconnaissance

First, identify the binary:

```bash
file ./tunnel
```

Output:

```text
ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
for GNU/Linux 3.2.0, not stripped
```

Important observations:

* 64-bit x86 ELF
* Dynamically linked
* PIE enabled
* **Not stripped**

The last point is particularly useful.

A stripped binary would remove most function names, making analysis harder. Here, useful symbols remain.

Check security properties:

```bash
checksec --file=./tunnel
```

Example:

```text
RELRO           Partial RELRO
Stack Canary    No canary found
NX              NX enabled
PIE             PIE enabled
```

For this challenge, these protections aren't particularly important because we aren't exploiting memory corruption. We're trying to understand the program's logic.

---

# 3. Quick Static Analysis

Before opening GDB, check the strings:

```bash
strings -a ./tunnel
```

Interesting strings include:

```text
/flag.txt
HTB{fake_flag_for_testing}
You break into the vault and read the secrets within...
Direction (L/R/F/B/U/D/Q)?
```

The presence of:

```text
/flag.txt
```

is immediately interesting.

It suggests that eventually the program probably opens and prints a flag file.

There is also:

```text
HTB{fake_flag_for_testing}
```

However, don't immediately assume this is the flag.

A useful RE principle:

> **A string existing in a binary doesn't mean the program actually uses it.**

We need to find the code referencing it.

---

# 4. Enumerating Functions

Because the binary isn't stripped:

```bash
nm -C ./tunnel
```

reveals useful functions:

```text
00000000000011b5 T get_cell
00000000000011e3 T prompt_and_update_pos
0000000000001446 T get_flag
0000000000001538 T main
00000000000020e0 R maze
```

This immediately gives us a rough picture:

```text
main
 ├── prompt_and_update_pos
 ├── get_cell
 └── get_flag
```

And most importantly:

```text
maze
```

is a symbol located in read-only data.

This is probably the map.

---

# 5. Understanding `get_cell()`

Open it in GDB:

```bash
gdb -q ./tunnel
```

Then:

```gdb
disas get_cell
```

The important assembly is:

```asm
0x11b5 <+0>:  mov    (%rdi),%eax
0x11b7 <+2>:  lea    (%rax,%rax,4),%rax
0x11bb <+6>:  lea    (%rax,%rax,4),%rax
0x11bf <+10>: shl    $0x4,%rax

0x11c3 <+14>: mov    0x4(%rdi),%edx
0x11c6 <+17>: lea    (%rdx,%rdx,4),%rdx
0x11ca <+21>: lea    (%rax,%rdx,4),%rax

0x11ce <+25>: mov    0x8(%rdi),%edx
0x11d1 <+28>: add    %rdx,%rax
0x11d4 <+31>: shl    $0x4,%rax

0x11d8 <+35>: lea    0xf01(%rip),%rdx
                         # 0x20e0 <maze>

0x11df <+42>: add    %rdx,%rax
0x11e2 <+45>: ret
```

We need to translate this into C-like logic.

---

# 6. Reverse Engineering the Index Calculation

The first coordinate is loaded:

```asm
mov (%rdi), %eax
```

Then:

```asm
lea (%rax,%rax,4), %rax
lea (%rax,%rax,4), %rax
```

Each operation multiplies by 5:

```text
x * 5
(x * 5) * 5
= x * 25
```

Then:

```asm
shl $0x4, %rax
```

means multiply by 16:

```text
x * 25 * 16
= x * 400
```

The second coordinate is multiplied by 20:

```text
y * 20
```

Then `z` is added.

Finally, the entire index is multiplied by 16.

Therefore:

```c
offset = (x * 400 + y * 20 + z) * 16;
```

and the base is:

```text
maze = 0x20e0
```

So:

```c
return maze + offset;
```

This tells us something extremely useful.

There are:

```text
20 × 20 × 20 = 8000
```

cells.

Each cell is:

```text
16 bytes
```

Therefore the maze occupies:

```text
8000 × 16 = 128000 bytes
```

---

# 7. Reconstructing the Cell Structure

Looking at the data with:

```gdb
x/128bx 0x20e0
```

reveals patterns like:

```text
00 00 00 00
00 00 00 00
00 00 00 00
00 00 00 00

00 00 00 00
00 00 00 00
01 00 00 00
01 00 00 00
```

The 16-byte structure is clearly four 32-bit integers.

We can reconstruct it as:

```c
struct Cell {
    int x;
    int y;
    int z;
    int type;
};
```

The fourth integer is particularly important.

The code accesses:

```asm
0xc(%rax)
```

which is offset `12` bytes into the structure.

Therefore:

```text
cell + 0xc = cell.type
```

---

# 8. Determining Cell Types

The maze can be extracted with Python.

```python
from pathlib import Path
import struct

data = Path("tunnel").read_bytes()

MAZE_OFFSET = 0x20e0
CELL_SIZE = 16

maze = {}

for i in range(20 * 20 * 20):
    offset = MAZE_OFFSET + i * CELL_SIZE

    x, y, z, cell_type = struct.unpack_from(
        "<4i",
        data,
        offset
    )

    maze[(x, y, z)] = cell_type

from collections import Counter

counts = Counter(maze.values())

for cell_type, count in sorted(counts.items()):
    print(f"type {cell_type}: {count}")
```

The result:

```text
type 0: 1
type 1: 62
type 2: 7936
type 3: 1
```

This is very informative.

There are:

```text
1 starting cell
62 normal tunnel cells
7936 walls
1 goal cell
```

Since:

```text
20 × 20 × 20 = 8000
```

everything is accounted for.

---

# 9. Finding the Start and Goal

The beginning of the maze contains:

```text
(0, 0, 0) type 0
```

and the only type-3 cell is:

```text
(19, 19, 19) type 3
```

So we have:

```text
START = (0, 0, 0)
GOAL  = (19, 19, 19)
```

But we should verify this by looking at `main()`.

---

# 10. Understanding `main()`

Run:

```gdb
disas main
```

Important instructions:

```asm
0x153d: movl $0x0,0x4(%rsp)
0x1545: movl $0x0,0x8(%rsp)
0x154d: movl $0x0,0xc(%rsp)
```

These initialize the three coordinates:

```c
x = 0;
y = 0;
z = 0;
```

So:

```text
Start = (0, 0, 0)
```

Then:

```asm
0x1569: call 0x11e3 <prompt_and_update_pos>
```

The program repeatedly asks for movement.

After each movement:

```asm
0x1571: call 0x11b5 <get_cell>
0x1576: cmpl $0x3,0xc(%rax)
0x157a: jne 0x155c
```

Translated:

```c
while (get_cell(&pos)->type != 3) {
    prompt_and_update_pos(&pos);
}
```

Therefore the program terminates the maze when we reach a cell with:

```text
type == 3
```

And our extracted maze has exactly one:

```text
(19, 19, 19)
```

So our interpretation is confirmed.

---

# 11. Reverse Engineering the Movement Function

Now examine:

```gdb
disas prompt_and_update_pos
```

The function:

1. Prints the prompt.
2. Reads one character.
3. Converts it to uppercase.
4. Uses a jump table to select the appropriate movement code.

The important portion is:

```asm
call __isoc99_scanf

call __ctype_toupper_loc

...

sub eax,0x42
cmp al,0x13
ja  ...
```

`0x42` is ASCII:

```text
0x42 = 'B'
```

So the program calculates:

```c
index = input - 'B';
```

Then:

```asm
lea rdx,[rip+0xe31]   # 0x2080
movsxd rax,DWORD PTR [rdx+rax*4]
add rax,rdx
jmp rax
```

That's a classic **switch statement implemented as a jump table**.

---

# 12. Understanding the Jump Table

The table is located at:

```text
0x2080
```

There are 20 entries.

Because the index is:

```c
input - 'B'
```

the entries correspond to:

```text
Index  Character

0      B
1      C
2      D
3      E
4      F
5      G
6      H
7      I
8      J
9      K
10     L
11     M
12     N
13     O
14     P
15     Q
16     R
17     S
18     T
19     U
```

Analyzing the jump targets gives:

```text
B → y--
D → z--
F → y++
L → x--
Q → quit
U → z++
```

The remaining characters mostly jump to a default/no-op case.

Therefore the real controls are:

```text
L = x - 1
R = no-op
F = y + 1
B = y - 1
U = z + 1
D = z - 1
Q = quit
```

An important lesson:

> Never trust the apparent meaning of an input character. Follow the jump table and the code it reaches.

In particular, `Q` really **does** mean quit. It was tempting to assume otherwise because of the movement letters, but the assembly settles the question.

---

# 13. Reconstructing the Program in C

At this point, the core of the program can be approximated as:

```c
struct Cell {
    int x;
    int y;
    int z;
    int type;
};

struct Position {
    int x;
    int y;
    int z;
};

struct Cell *get_cell(struct Position *p) {
    int index =
        p->x * 400 +
        p->y * 20 +
        p->z;

    return &maze[index];
}
```

And movement is approximately:

```c
switch (toupper(input)) {

    case 'L':
        x--;
        break;

    case 'R':
        x++;
        break;

    case 'F':
        y++;
        break;

    case 'B':
        y--;
        break;

    case 'U':
        z++;
        break;

    case 'D':
        z--;
        break;

    case 'Q':
        puts("Goodbye!");
        exit(-2);
}
```

There are also boundary checks and wall checks around each movement.

The maze-solving problem has now become a normal graph-search problem.

---

# 14. Converting the Maze into a Graph

Each non-wall cell is a node.

From each node we can move to up to six neighboring cells:

```text
             U
             |
             |
       L ----+---- R
             |
             |
             D
```

And along the third dimension:

```text
F / B
```

More formally:

```text
L = (-1,  0,  0)
R = (+1,  0,  0)

B = ( 0, -1,  0)
F = ( 0, +1,  0)

D = ( 0,  0, -1)
U = ( 0,  0, +1)
```

A movement is valid when:

1. The resulting coordinate remains inside `0..19`.
2. The destination cell is not type `2`.

---

# 15. Solving with BFS

Because every movement costs exactly one command, **Breadth-First Search (BFS)** is a natural solution.

BFS guarantees the shortest path in an unweighted graph.

The solver:

```python
from collections import deque
from pathlib import Path
import struct

data = Path("tunnel").read_bytes()

MAZE_OFFSET = 0x20e0
CELL_SIZE = 16

MOVES = {
    "L": (-1,  0,  0),
    "R": ( 1,  0,  0),
    "B": ( 0, -1,  0),
    "F": ( 0,  1,  0),
    "D": ( 0,  0, -1),
    "U": ( 0,  0,  1),
}

maze = {}

for i in range(20 * 20 * 20):
    offset = MAZE_OFFSET + i * CELL_SIZE

    x, y, z, cell_type = struct.unpack_from(
        "<4i",
        data,
        offset
    )

    maze[(x, y, z)] = cell_type

start = (0, 0, 0)
goal = (19, 19, 19)

queue = deque([start])

parent = {
    start: None
}

move_used = {}

while queue:

    pos = queue.popleft()

    if pos == goal:
        break

    x, y, z = pos

    for command, (dx, dy, dz) in MOVES.items():

        nxt = (
            x + dx,
            y + dy,
            z + dz
        )

        # Bounds check
        if not (
            0 <= nxt[0] < 20 and
            0 <= nxt[1] < 20 and
            0 <= nxt[2] < 20
        ):
            continue

        # Wall
        if maze[nxt] == 2:
            continue

        # Already visited
        if nxt in parent:
            continue

        parent[nxt] = pos
        move_used[nxt] = command

        queue.append(nxt)


if goal not in parent:
    print("No path found!")
    exit()


# Reconstruct path
path = []

pos = goal

while parent[pos] is not None:

    path.append(move_used[pos])

    pos = parent[pos]

path.reverse()

path = "".join(path)

print("Start:", start)
print("Goal :", goal)
print("Path :", path)
print("Length:", len(path))
```

The resulting route was:

```text
UUURFURURRFRRFFUUFURRUFUFFRFUFUUUUFFRRUUUFURFDFFUFFRRRRRFRR
```

Length:

```text
59
```

---

# 16. Testing Locally

Before touching the remote service, always test the solution against the local binary.

This is a useful general CTF habit:

> **Validate your theory locally before attacking the remote instance.**

Run:

```bash
printf '%s' \
'UUURFURURRFRRFFUUFURRUFUFFRFUFUUUUFFRRUUUFURFDFFUFFRRRRRFRR' \
| ./tunnel
```

The binary reaches the vault:

```text
You break into the vault and read the secrets within...
```

and prints:

```text
HTB{fake_flag_for_testing}
```

This confirms:

* The maze was parsed correctly.
* The coordinate system was correct.
* The movement directions were correct.
* The BFS path was correct.
* The goal condition was correct.

The fake flag is simply the local/test value embedded in the binary.

---

# 17. Understanding `get_flag()`

The final function is:

```gdb
disas get_flag
```

The important instructions include:

```asm
lea ... # 0x2042
lea ... # 0x2044
call fopen@plt
```

The strings around that area are:

```text
"r"
"/flag.txt"
```

Then the function eventually calls:

```asm
fgets
puts
fclose
```

So conceptually:

```c
FILE *f = fopen("/flag.txt", "r");

fgets(buffer, 0x80, f);

puts(buffer);

fclose(f);
```

This explains why the local binary gives us:

```text
HTB{fake_flag_for_testing}
```

The local challenge environment contains a test `/flag.txt`.

The remote service has the real one.

---

# 18. Getting the Remote Flag

Send the exact same path to the remote service:

```bash
printf '%s' \
'UUURFURURRFRRFFUUFURRUFUFFRFUFUUUUFFRRUUUFURFDFFUFFRRRRRFRR' \
| nc 154.57.164.80 31062
```

The remote instance responds with:

```text
You break into the vault and read the secrets within...
HTB{tunn3l1ng_ab0ut_in_3d_e5cef73b850211b7b9fc504adc4820d0}
```

Therefore the flag is:

```text
HTB{tunn3l1ng_ab0ut_in_3d_e5cef73b850211b7b9fc504adc4820d0}
```

---

# 19. Full RE Chain

The entire solve can be summarized as:

```text
                 ./tunnel
                    │
                    ▼
             strings / nm
                    │
                    ├── get_cell()
                    ├── prompt_and_update_pos()
                    ├── get_flag()
                    └── maze
                    │
                    ▼
              Analyze get_cell()
                    │
                    ▼
          20 × 20 × 20 maze
          16-byte cells
                    │
                    ▼
           struct Cell {
             int x;
             int y;
             int z;
             int type;
           }
                    │
                    ▼
             Analyze main()
                    │
             ┌──────┴──────┐
             ▼             ▼
       Start=(0,0,0)   Goal=type 3
                           │
                           ▼
                     (19,19,19)
                           │
                           ▼
            Analyze movement function
                           │
                           ▼
                 Recover jump table
                           │
                           ▼
                L/R/F/B/U/D movement
                           │
                           ▼
              Extract maze from binary
                           │
                           ▼
                    BFS shortest path
                           │
                           ▼
       U U U R F U R U R R F R R ...
                           │
                           ▼
                  Reach type 3
                           │
                           ▼
                    get_flag()
                           │
                           ▼
                  /flag.txt
                           │
                           ▼
                       FLAG
```

---

# 20. Important Reverse Engineering Lessons

## 20.1 Start with reconnaissance

Don't immediately start stepping instruction-by-instruction.

First:

```bash
file ./binary
checksec --file=./binary
strings -a ./binary
nm -C ./binary
```

These commands quickly tell you:

* What kind of binary you're dealing with.
* Whether symbols remain.
* Whether useful strings exist.
* What functions/data might be interesting.

---

## 20.2 Function names are extremely valuable

Because this binary wasn't stripped, names such as:

```text
get_cell
get_flag
prompt_and_update_pos
main
maze
```

gave us a huge advantage.

If symbols exist, inspect them before blindly reversing the entire binary.

---

## 20.3 Learn to translate assembly into C

For example:

```asm
mov (%rdi), %eax
```

can be thought of as:

```c
eax = *(int *)rdi;
```

And:

```asm
mov 0x4(%rdi), %edx
```

means:

```c
edx = *(int *)(rdi + 4);
```

This lets us recognize:

```text
offset 0  → x
offset 4  → y
offset 8  → z
offset 12 → type
```

That is how we reconstructed the structure.

---

## 20.4 LEA isn't only for addresses

This:

```asm
lea (%rax,%rax,4), %rax
```

is equivalent to:

```text
rax = rax + rax*4
```

or:

```text
rax *= 5
```

Reverse engineers frequently use this trick to identify compiler-generated multiplication.

For example:

```asm
lea (%rax,%rax,4), %rax
lea (%rax,%rax,4), %rax
```

means:

```text
rax *= 5
rax *= 5
```

so:

```text
rax *= 25
```

---

## 20.5 Shift instructions are often multiplication

For positive integers:

```asm
shl $4, %rax
```

means:

```text
rax *= 16
```

Likewise:

```asm
shl $3
```

means multiplication by 8.

Combining these operations is often enough to reconstruct array indexing.

---

## 20.6 Recognize jump tables

This pattern:

```asm
sub eax, 0x42
cmp al, 0x13
ja ...
lea rdx, ...
movsxd rax, DWORD PTR [rdx+rax*4]
add rax, rdx
jmp rax
```

is a classic compiled `switch`.

Once you recognize it, you can reconstruct:

```c
switch (input) {
    ...
}
```

without manually following every branch.

---

## 20.7 Don't trust the UI

The displayed prompt was:

```text
Direction (L/R/F/B/U/D/Q)?
```

It's easy to assume:

```text
Q = quit
R = right
```

But the actual behavior must come from the code.

In this challenge:

```text
Q = quit
```

was confirmed by following the jump table.

The lesson is broader:

> **The binary is the source of truth.**

---

## 20.8 Data structures are often easier to understand than code

Once we discovered:

```text
maze @ 0x20e0
```

and:

```text
(x * 400 + y * 20 + z) * 16
```

the challenge became much easier.

Instead of manually exploring 8000 cells through GDB, we could directly extract the data from the executable.

Whenever you see a large static data region, ask:

```text
Is this a table?
An array?
A structure?
A lookup table?
An encoded blob?
```

---

# 21. Useful GDB Commands for Similar Challenges

### Start GDB

```bash
gdb -q ./binary
```

### List functions

```gdb
info functions
```

or:

```bash
nm -C ./binary
```

### Disassemble a function

```gdb
disas main
```

or:

```gdb
disas get_flag
```

### Examine memory as hex

```gdb
x/64bx ADDRESS
```

### Examine memory as integers

```gdb
x/32wx ADDRESS
```

### Examine strings

```gdb
x/s ADDRESS
```

### Examine instructions

```gdb
x/20i ADDRESS
```

### Break at a function

```gdb
break main
```

### Break at an address

For a PIE binary, prefer a symbolic/function-relative breakpoint or a runtime address after the binary loads.

For example:

```gdb
break *prompt_and_update_pos+74
```

rather than blindly using:

```gdb
break *0x1430
```

because PIE means the executable's runtime base address is relocated.

### Run

```gdb
run
```

### Continue

```gdb
continue
```

### Inspect registers

```gdb
info registers
```

### Inspect a specific register

```gdb
p/x $rax
```

### Inspect memory pointed to by a register

```gdb
x/4wx $rax
```

---

# 22. General Workflow for Similar RE CTF Challenges

When given an unknown ELF binary, a useful workflow is:

```text
1. file
      ↓
2. checksec
      ↓
3. strings
      ↓
4. nm / readelf
      ↓
5. identify interesting functions
      ↓
6. disassemble main()
      ↓
7. identify important helper functions
      ↓
8. translate assembly into pseudocode
      ↓
9. identify data structures
      ↓
10. inspect .rodata/.data
      ↓
11. write a parser if necessary
      ↓
12. reproduce logic in Python
      ↓
13. solve algorithmically
      ↓
14. validate locally
      ↓
15. send solution remotely
```

The key mindset is:

> **Don't solve the interface. Solve the underlying program.**

---

# 23. Final Commands

### Extract the maze

```bash
python3 solve.py
```

### Test locally

```bash
printf '%s' \
'UUURFURURRFRRFFUUFURRUFUFFRFUFUUUUFFRRUUUFURFDFFUFFRRRRRFRR' \
| ./tunnel
```

### Get the remote flag

```bash
printf '%s' \
'UUURFURURRFRRFFUUFURRUFUFFRFUFUUUUFFRRUUUFURFDFFUFFRRRRRFRR' \
| nc 154.57.164.80 31062
```

---

# Flag

```text
HTB{tunn3l1ng_ab0ut_in_3d_e5cef73b850211b7b9fc504adc4820d0}
```

---

# Key Takeaways

The most important things to remember from this challenge are:

* Use `file`, `checksec`, `strings`, and `nm` before deep analysis.
* Function names can reveal the intended architecture of a binary.
* Learn to convert assembly instructions into C-like operations.
* `lea` is frequently used for arithmetic, not just addresses.
* Shifts can reveal multiplication.
* Jump tables usually represent compiled `switch` statements.
* Static data can often be parsed directly instead of explored manually.
* Identify structures from repeated offsets.
* Use GDB to confirm hypotheses rather than guessing.
* When a binary contains a large map/table, write a Python extractor.
* Turn maze/path problems into graph problems.
* BFS is ideal when every movement has equal cost.
* Always test locally before attacking the remote service.
* **Trust the code, not the UI.**
* Most importantly: once you can reconstruct what the compiler produced, you don't need to "hack" the binary—you can simply reproduce its logic.
