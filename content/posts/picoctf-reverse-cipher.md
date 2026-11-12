---
title: "picoCTF: Reverse Cipher"
date: 2026-11-12
draft: false
tags: ["rev", "picoctf", "ghidra", "python"]
---

A text file holds an encrypted flag and a binary encoded it. Reading the encoder in Ghidra shows the rule, and inverting it in Python gives the flag.

<!--more-->

**CTF:** picoCTF\
**Category:** Reverse Engineering\
**Tools:** Ghidra, Python\
**Given:** an encoding binary and a text file with the encrypted flag

## Challenge

We get a binary and a text file containing the encrypted flag:

```
academy{w1{1wq87<284;2<}
```

The goal is to reverse the encoding and recover the real flag.

## Approach

I opened the binary in Ghidra. The decompiled code of the encoder looks like this:

![Ghidra decompilation of the encoding loops](/img/posts/reverse-cipher-ghidra.png)

Here is what it does:

1. It opens `flag.txt` for reading and `rev_this` for appending (this is where the encrypted flag is written), then reads 24 bytes from `flag.txt`.
2. **First loop** (`i = 0` to `7`): each character is written to the output unchanged. So the first **8 characters** are preserved, which is exactly the readable prefix `academy{`.
3. **Second loop** (`j = 8` to `22`): the condition `(j & 1) == 0` tests whether the index is even.
   - **even index**: the character is incremented by **5**
   - **odd index**: the character is changed by **-2**

Neither loop touches index 23, and the encrypted text still ends with the closing `}`.

To decode, we simply apply the opposite operation at each index:

| Index (8 to 22) | Encoder | Decoder |
|---|---|---|
| even | +5 | **-5** |
| odd | -2 | **+2** |

Example with the first two encoded characters:

- index 8 (even): `w` is 119, and 119 - 5 = 114, which is `r`
- index 9 (odd): `1` is 49, and 49 + 2 = 51, which is `3`

The text already starts to look like leetspeak ("r3v3rs...").

## Solution

```python
text = "academy{w1{1wq87<284;2<}"

flag = list(text[:8])                       # first 8 characters are unchanged
for i in range(8, len(text) - 1):           # indices 8 to 22
    if i % 2 == 0:
        flag.append(chr(ord(text[i]) - 5))  # undo the +5
    else:
        flag.append(chr(ord(text[i]) + 2))  # undo the -2
flag.append(text[-1])                       # closing brace

print("".join(flag))
```

Output:

```
academy{r3v3rs397436647}
```

**Flag:** `academy{r3v3rs397436647}`

## Takeaways

- Ghidra's decompiler makes this kind of logic easy to read: a parity check like `(j & 1) == 0` tells you right away that even and odd positions are treated differently.
- A readable prefix is a quick hint that part of the data is not encoded.
- To reverse a per-character cipher, apply the inverse operation at each index.
- My first solver stopped one character early and dropped the closing `}`. Always check that the output matches the expected flag format.
