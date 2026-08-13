---
layout: post
title: "Buffer Overflow Basics — Stack Layout & gets()"
date: 2026-08-12 09:00:00 -0800
categories: [pwn]
tags: [buffer-overflow, x86-64, stack, gdb]
platform: picoCTF
points: 100
---

## Challenge Info

- **Platform:** picoCTF
- **Category:** Binary Exploitation
- **Points:** 100
- **Tools used:** GDB, picoCTF primer (primer.cylabacademy.org)

## The Challenge

A binary that reads user input into a fixed-size stack buffer using `gets()`,
with no bounds checking. The goal is to understand how overflowing that
buffer can overwrite the saved return address and redirect execution.

## Initial Approach

Started by looking at the disassembly in GDB to understand the function
prologue/epilogue and how the stack frame is laid out — where the buffer
sits relative to the saved base pointer (`rbp`) and return address.

## The Vulnerability / Concept

`gets()` reads input with no length limit, so it will happily write past the
end of a fixed-size buffer. Because the stack grows downward but buffers are
filled upward (low to high addresses), writing enough bytes past the buffer
overwrites adjacent stack data — including the saved return address that
`ret` uses to resume execution in the caller.

If you control that overwritten value precisely, you control where the CPU
jumps next after the function returns. This is the whole basis of classic
stack-based buffer overflow exploitation.

## Solution

1. Disassemble the vulnerable function in GDB, find the buffer's offset from
   `rbp` and the saved return address.
2. Calculate padding needed: `offset_to_return_address = buffer_size + 8`
   (the extra 8 bytes account for the saved `rbp` on x86-64).
3. Craft input: padding bytes + target address to overwrite the return
   address.

```bash
gdb ./vulnerable_binary
(gdb) disas main
(gdb) info frame
```

## What I'd Do Differently

Would've saved time by checking for a stack canary first (`checksec` or
looking for canary-related instructions in the disassembly) before assuming
a straightforward overwrite would work — canaries change the approach
entirely.

## Flag

`picoCTF{...}` <!-- fill in your actual flag -->
