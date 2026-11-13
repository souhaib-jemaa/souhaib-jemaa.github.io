---
title: "ret2 wargames deusx64, Level 1: x64 ASM"
date: 2026-11-13
draft: false
tags: ["rev", "asm", "x86-64", "ret2-wargames"]
---

Level 1 of the ret2 wargames deusx64 track has three mini-challenges. The first one, x64 ASM, asks for two arguments that make a short assembly function return 31337.

<!--more-->

**CTF:** ret2 wargames (deusx64)\
**Category:** Reverse Engineering\
**Level:** 1\
**Mini-challenge:** x64 ASM

## Challenge

This level is a job application: a "legacy developer" posting asks us to complete a skills assessment made of three mini-challenges, **x64 ASM**, **SERIAL** and **ENCRYPT**.

![Job posting showing the three mini-challenges](/img/posts/ret2-deusx64/1.png)

The first mini-challenge, **x64 ASM**, gives us a short function in Intel syntax and asks us to supply arguments so that it returns `31337`.

![The x64 ASM challenge and its function code](/img/posts/ret2-deusx64/2.png)

## Approach

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

## Solution

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


