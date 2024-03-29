---
layout: post
title: x86-64 Assembly Cheat Sheet
tags: [cheat sheet, assembly]
---

This is still being updated upon I find useful things.

## Registers

### General-Purpose Registers

| qword | dword | word | high byte | low byte |               |
| ----- | ----- | ---- | --------- | -------- | ------------- |
| rax   | eax   | ax   | ah        | al       |               |
| rbx   | ebx   | bx   | bh        | bl       |               |
| rcx   | ecx   | cx   | ch        | cl       |               |
| rdx   | edx   | dx   | dh        | dl       |               |
| rbp   | ebp   | bp   | bph       | bpl      | base pointer  |
| rsp   | esp   | sp   | sph       | spl      | stack pointer |
| rsi   | esi   | si   | sih       | sil      |               |
| rdi   | edi   | di   | dih       | dil      |               |
| r8    | r8d   | r8w  |           |          |               |
| ...   | ...   | ...  |           |          |               |
| r15   | r15d  | r15w |           |          |               |

### Calling Convention

In [64-bit System V](https://riptutorial.com/x86/example/11197/64-bit-system-v):

| Argument | 1   | 2   | 3   | 4   | 5  | 6  | 7   | 8   | 9     | 10      | ... | return |
| -------- | --- | --- | --- | --- | -- | -- | --- | --- | ----- | ------- | --- | ------ |
| Location | rdi | rsi | rdx | rcx | r8 | r9 | r10 | r11 | [rsp] | [rsp+8] | ... | rax    |

Preserved registers: values of `r12-r15`, `rbx`, `rsp`, `rbp` are preserved across function calls.

### System Call Convention

Linux syscall follows a slightly different convention to function calls (see `man syscall`):

In x86-64, the syscall number is passed using `rax`, and return value is stored in `rax`, and the registers used to pass the arguments are listed in the table below:

| Argument | 1   | 2   | 3   | 4   | 5  | 6  | return |
| -------- | --- | --- | --- | --- | -- | -- | ------ |
| Location | rdi | rsi | rdx | r10 | r8 | r9 | rax    |

[List of linux syscalls](https://filippo.io/linux-syscall-table/).

## Conditional Jump Instructions

[List of conditional jumps](http://unixwiz.net/techtips/x86-jumps.html).

## More Resources

- [`Ike: The Systems Hacking Handbook](https://ike.mahaloz.re/)