---
title: "Securinets Tekup: elimination_project"
date: 2026-11-23
draft: false
tags: ["rev", "ghidra", "ida", "patching", "anti-debugging", "python", "securinets"]
---

A countdown window asks for a deactivation key, and every wrong answer costs you 15 seconds. My first idea was the lazy one: patch the program so it always says yes.

<!--more-->

**CTF:** Securinets Tekup\
**Category:** Reverse Engineering\
**Challenge:** elimination_project


When I run the program, a window pops up with a countdown called "time until detonation" and a box asking for a deactivation key. I type something and press Enter. If the key is wrong, I lose time, and when the timer hits zero, it's game over.

![The elimination_project window with the countdown and the key prompt](/img/posts/tekup_elim_operation/1.png)


Before opening any disassembler, I ran `strings` on the binary to see what text is hiding inside it. Two lines caught my eye:

![Output of strings showing the penalty and flag messages](/img/posts/tekup_elim_operation/2.png)

The first one, `ACCESS DENIED - PENALTY: -15s`, explains the time we lose after each wrong key. The second one, `SEQUENCE ABORTED. FLAG: Securinets{%s}`, is the message the program prints when we get it right. The `%s` is just a blank that the program fills in at the last moment.

So the program prints the flag for us once it's happy with our key. That gave me a lazy idea: if I can't find the key, why not trick the program into being happy anyway?

## The lazy plan: patch it

I opened the binary in Ghidra and looked for the place where the program checks my key. It calls a function, and right after that there is a small test: did the check say yes? If not, the program jumps to the "wrong key" part and takes 15 seconds off the clock.

Here is that part of the code before my patch. The `JZ` line is the jump to the wrong-key part:

![Ghidra listing before the patch](/img/posts/tekup_elim_operation/before_patch.png)

My plan was simple: replace that jump with a NOP, which is an instruction that does nothing. Without the jump, the program can never go down the wrong-key road, so it should always believe my key is right and print the flag. Here is the code after the patch:

![Ghidra listing after the patch](/img/posts/tekup_elim_operation/after_patch.png)

## It worked... sort of

I ran the patched program and typed a joke message, "IT WAS A NICE TRY THOUGH :)". The timer stopped, the window turned green, and the flag message showed up. But look at what the flag says:

![The patched program accepting any key and printing it back](/img/posts/tekup_elim_operation/3.png)

The program just prints back whatever I typed! The blank in `Securinets{%s}` gets filled with my own input, so skipping the check only gives me a fake flag made of my own words. The real flag must be the key that the program actually accepts.

So patching is not enough. I have to understand how the program checks the key and work out the right one myself.

Time to understand how the program checks the key. I opened the binary in IDA and found the function that tests it. Here it is:

![The function that tests the key](/img/posts/tekup_elim_operation/4.png)

It does three things. First, it makes sure the key is exactly 32 characters long, otherwise it gives up right away. Then it prepares the random number generator with the current time, but it immediately changes the seed to this:

```
dword_41280 & 0x1337BEEF ^ sub_2583(v4)
```

Finally, for each of the 32 characters, it draws a random number, mixes it with the character, scrambles the bits and compares the result with a hidden value. The key is accepted only if every single comparison matches perfectly.

So everything depends on those random numbers, and to predict them I need to know the seed. That means figuring out two things: `dword_41280` and `sub_2583`.

## The anti-debugging trick

I started with `dword_41280`. Following its cross references shows who touches it:

![Cross references of dword_41280](/img/posts/tekup_elim_operation/5.png)

One function writes to it and one reads it (the test function). The one that writes looks like this:

![The function that writes dword_41280](/img/posts/tekup_elim_operation/6.png)

It calls `ptrace(PTRACE_TRACEME, 0, 0, 0)`. When the program runs normally, this returns 0. But if something is already tracing the program, like a debugger, it returns -1. That result is stored in `dword_41280`, which then gets mixed into the seed.

So this is an anti-debugging trick: run the program in a debugger and the seed changes, and nothing works anymore. When the program runs normally, `dword_41280` is simply 0.

## The scary function

Next is `sub_2583`, which gives the other half of the seed:

![The sub_2583 function](/img/posts/tekup_elim_operation/7.png)

It looks heavy: time, random numbers, sines and cosines, big math. I translated it to Python and ran it many times, and it always gave back the same value, `0xF82A`. That makes sense once you look closer: most of those operations are just noise that always ends up as 1, like sin² + cos² or cosh² - sinh². Whatever the time or the random numbers are, everything collapses into the same constant.

Here is my translation:

```python
import ctypes
import math
import time
libc = ctypes.CDLL("libc.so.6")

MASK_64 = 0xFFFFFFFFFFFFFFFF
MASK_32 = 0xFFFFFFFF

def sub_2474(a1: int) -> int:
    a1 &= MASK_64
    a1 ^= (a1 >> 33)
    a1 = (a1 * 0xFF51AFD7ED558CCD) & MASK_64
    a1 ^= (a1 >> 33)
    a1 = (a1 * 0xC4CEB9FE1A85EC53) & MASK_64
    a1 ^= (a1 >> 33)
    return a1

def sub_24D2(a1: int, a2: int, a3: int) -> int:
    return pow(a1, a2, a3)

def sub_2545(a1: int) -> int:
    while a1 != 1:
        if a1 & 1:
            a1 = 3 * a1 + 1
        else:
            a1 >>= 1
    return 1

def sub_2583() -> int:
    v0 = int(time.time())
    v19 = v0 ^ (time.process_time_ns() // 1000)

    v18 = libc.rand() % 900000000 + 1000000
    v17 = libc.rand() % 999999 + 2
    x = float(libc.rand() % 90) / 10.0 + 0.5

    v15 = sub_2474((v19 + v18) & MASK_64)
    v14 = math.cos(x) ** 2 + math.sin(x) ** 2
    v13 = math.cosh(x) ** 2 - math.sinh(x) ** 2
    v12 = math.exp(math.log(x) - math.log(x))

    v11 = sub_24D2(v17, 1000000006, 1000000007)
    v10 = sub_2545((v18 ^ (v19 & 0xFFFF)) | 1)

    v18_32 = ctypes.c_int32(v18).value
    diff = ctypes.c_int32(v18_32 * v18_32 - (v18_32 - 1) * (v18_32 + 1)).value

    part1 = ctypes.c_int64(diff * 1597463007 * round(v14 * v13 * v12)).value
    part2 = ctypes.c_int64(v10 * 1597481461 * v11).value

    even_check = -(((v15 & 0xFF) * ((v15 + 1) & 0xFF)) & 1)
    part3 = (v15 % 0x26F5) & (even_check & MASK_64)

    result = ((part1 ^ part2) + part3 + v10 - v11) & MASK_64
    return result


print(sub_2583())
```

So when the program runs normally, the seed is `0 ^ 0xF82A`, which is simply **0xF82A**.

## Brute force

Now that the seed is fixed, the "random" numbers are always the same ones, in the same order. For each position of the key, the program wants this to be exactly 0:

```
sub_28F2(v7, (v8 >> 8) & 7) ^ *(v6 + i)
```

So for each position, I ask the random generator for the number the program would get, then I try all 256 possible characters and keep the one that makes the result 0. I did this by translating the C code to Python.

One catch: the random generator of C is not the same as the one in Python, so my script calls the real C one through `ctypes`.

```python
import ctypes
import time
import struct
libc = ctypes.CDLL("libc.so.6")


def sub_28F2(a1: int, a2: int) -> int:
    a1 &= 0xFF
    shift = a2 & 7
    return ((a1 << shift) | (a1 >> ((8 - shift) & 7))) & 0xFF

v6 = [
    0x010462B9D7A02B54,
    0x556E6C337EBAB7E1,
    0x2ACE3EF6420E861B,
    0xB547414EB2B6BFF2
]

v6_bytes = struct.pack("<4Q", *v6) # because the binary is LSB (LE)

flag=[]
libc.srand(0xF82A)

for i in range(32):
    v8=libc.rand()
    for char in range(256):
        v7 = (char ^ v8) & 0xFF
        if (sub_28F2(v7, (v8 >> 8) & 7) ^ v6_bytes[i])==0:
            flag.append(chr(char))
            break

print("".join(flag))
```
