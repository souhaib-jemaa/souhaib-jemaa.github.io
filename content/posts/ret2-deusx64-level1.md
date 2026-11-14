---
title: "ret2 wargames deusx64: Level 1"
date: 2026-11-13
draft: false
tags: ["rev", "asm", "x86-64", "ret2-wargames"]
---

Level 1 of the ret2 wargames deusx64 track is a job application made of three mini-challenges: x64 ASM, SERIAL and ENCRYPT.

<!--more-->

**CTF:** ret2 wargames (deusx64)\
**Category:** Reverse Engineering\
**Level:** 1\
**Mini-challenges:** x64 ASM, SERIAL, ENCRYPT

## Overview

This level is a job application: a "legacy developer" posting asks us to complete a skills assessment made of three mini-challenges, **x64 ASM**, **SERIAL** and **ENCRYPT**.

![Job posting showing the three mini-challenges](/img/posts/ret2-deusx64/1.png)

## 1. x64 ASM

### Challenge

The first mini-challenge gives us a short function in Intel syntax and asks us to supply arguments so that it returns `31337`.

![The x64 ASM challenge and its function code](/img/posts/ret2-deusx64/2.png)

### Approach

The function follows the System V AMD64 calling convention, so the first argument is in `rdi`, the second one is in `rsi`, and the return value is in `rax`. Let `x` be the first argument and `y` the second one.

```
push    rdi
push    rsi
pop     rdi
pop     rsi
xor     rax, rax
mov     rax, rsi
add     rax, rsi
add     rax, $1024
sub     rdi, $256
add     rax, rdi
ret
```

Let's follow the registers step by step:

| Instruction | Effect |
|---|---|
| `push rdi`, `push rsi`, `pop rdi`, `pop rsi` | swaps the arguments: `rdi = y`, `rsi = x` |
| `xor rax, rax` | `rax = 0` |
| `mov rax, rsi` | `rax = x` |
| `add rax, rsi` | `rax = 2x` |
| `add rax, $1024` | `rax = 2x + 1024` |
| `sub rdi, $256` | `rdi = y - 256` |
| `add rax, rdi` | `rax = 2x + 1024 + y - 256` |
| `ret` | returns `rax` |

The swap at the start is easy to miss, and it matters: because of it, it is the **first** argument that gets doubled.

### Solution

We need the return value to be 31337, so we solve:

```
31337 = 2*x + 1024 + y - 256
2*x + y = 30569
```

This equation has many solutions. The simplest one is to set `x = 0`, which gives `y = 30569`:

- **argument 1 = 0**
- **argument 2 = 30569**

Check: 2 * 0 + 1024 + 30569 - 256 = 31337.

Any other pair with `2x + y = 30569` works too, for example `x = 1` and `y = 30567`.

## 2. SERIAL

### Challenge

When we choose option `2` (SERIAL), the program asks us for a valid serial number.

![The serial check prompt](/img/posts/ret2-deusx64/3.png)

### Approach

The challenge gives us the source code of the function that runs when we choose SERIAL, `start_serial_test()`:

![Source code of start_serial_test](/img/posts/ret2-deusx64/4.png)

It reads our input with `fgets`, removes the trailing newline, and then calls `check_serial(serial)`. The value it returns is the number of valid characters: the serial is accepted only if it is **6**, otherwise the program prints how many characters were valid ("N OF 6 WERE VALID").

We don't have the source code of `check_serial`, only its disassembly:

![Disassembly of check_serial](/img/posts/ret2-deusx64/5.png)

Here is how it works:

1. It sets a counter `valid` to 0, then calls `strlen` on our input and compares the result with `0x6`. **The serial must be exactly 6 characters long**, otherwise the function jumps to a branch that prints an error message in red.
2. Then it checks the characters **one at a time**, comparing each one with a hardcoded byte. For example, the first character is loaded with `movzx eax, byte [rax]` and compared with `0x31` (`cmp al, 0x31`). `0x31` is 49 in decimal, which is the ASCII code of the character `1`.
3. If the two are equal, the counter is incremented (`add dword [valid], 0x1`). If not, `jne` skips the increment.
4. The same comparison is repeated for the other characters.

### Solution

Reading the six hardcoded bytes and converting them from ASCII gives us the serial:

| Position | Byte | Character |
|---|---|---|
| 1 | `0x31` | `1` |
| 2 | `0x38` | `8` |
| 3 | `0x39` | `9` |
| 4 | `0x37` | `7` |
| 5 | `0x31` | `1` |
| 6 | `0x35` | `5` |

The valid serial is **`189715`**.

Note: because the program prints how many characters are valid, the serial could also be brute-forced one position at a time without reading the disassembly.
