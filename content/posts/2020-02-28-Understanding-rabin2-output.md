---
title:  "Understanding rabin2 output"
date:   2020-02-28 00:00:00
author: "David"
tags:
    - Reverse Engineering
    - Binary Analysis
---

## Prelude
If you don't know what rabin2 is/what it does.
> Rabin2 understands many file formats: Java CLASS, ELF, PE, Mach-O or any format supported by plugins, and it is able to obtain symbol import/exports, library dependencies, strings of data sections, xrefs, entrypoint address, sections, architecture type. [[src]](https://radare.gitbooks.io/radare2book/tools/rabin2/intro.html)

The binary info option of rabin2 outputs quite a lot of information, however there's no explanation to what each of the values mean, they can be quite cryptic especially to those not familiar with reverse engineering. I tried searching around but couldn't find any information regarding it so I made this table to help with interpretting the values, not all values are included here, but I added all that I could figure out so far, this may be updated in the future.

## Table

| Header | Explanation | Remark |
|---|---|---|
| arch | Architecture of the binary (Eg. ARM, x86) | |
| baddr | Base Address, used to calculate the absolute address when the program is loaded in memory. | |
| laddr | Load Address | [Reference](https://reverseengineering.stackexchange.com/a/19783) |
| bits | Size of address pointer of program | |
| bintype | The type of binary (Eg. PE, ELF), blank if not a known binary type | |
| linenum | [COFF Line Numbers](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#coff-line-numbers-deprecated) for PE or DWARF in ELF, Debugging line numbers relating to the source code. | |
| lsyms | Whether the binary contains debug symbols. Having symbols allows you to see function and variable names. | |
| endian | Endianness of the binary (Little or Big) | |
| binsz | Size of the binary in bytes | |
| Canary | Stack canary. A random value is placed on the stack at the start of the function, this value is checked for modification before a function returns because it has to be overwritten in order to overwrite the return pointer. | Protection Mechanism |
| retguard | Similar function to stack canary. [More info](https://isopenbsdsecu.re/mitigations/retguard/) | Protection Mechanism |
| sanitiz | Address Sanitizer (ASAN) a memory error detector for C/C++ | Should only be seen for debug builds because of the performance impact of ASAN |
| NX | No execute bit. W^X -> Memory regions cannot be both writable and executable | Protection Mechanism |
| PIC | Position Independent Code, allows ASLR | Protection Mechanism |
| reloc | Performs Load-time relocation | |
| Relro | [Makes some binary sections read-only.](https://ctf101.org/binary-exploitation/relocation-read-only/) | Protection Mechanism |
| rpath | The run-time library search path hard-coded in an executable file or library. | |
| Signed | Digitally signed | Only for PE binaries |
| Static | Whether the binary is statically linked | |
| Stripped | Whether the binary contain debug information | |
| va | Uses virtual addressing | |

## Sample outputs

```
$ rabin2 -I /bin/bash
arch     x86
baddr    0x0
binsz    1111705
bintype  elf
bits     64
canary   true
sanitiz  false
class    ELF64
crypto   false
endian   little
havecode true
intrp    /lib64/ld-linux-x86-64.so.2
laddr    0x0
lang     c
linenum  false
lsyms    false
machine  AMD x86-64 architecture
maxopsz  16
minopsz  1
nx       true
os       linux
pcalign  0
pic      true
relocs   false
relro    full
rpath    NONE
static   false
stripped true
subsys   linux
va       true
```

```
$ rabin2 -I /mnt/c/Windows/System32/ipconfig.exe
arch     x86
baddr    0x140000000
binsz    34816
bintype  pe
bits     64
canary   false
retguard false
sanitiz  false
class    PE32+
cmp.csum 0x0000cef7
compiled Tue Jan 14 03:35:17 1986
crypto   false
dbg_file ipconfig.pdb
endian   little
havecode true
hdr.csum 0x0000cef7
guid     FF8C0F8EBC5D9AA01B9260167EE2FC3C1
laddr    0x0
linenum  false
lsyms    false
machine  AMD 64
maxopsz  16
minopsz  1
nx       true
os       windows
overlay  false
pcalign  0
pic      true
relocs   false
signed   false
static   false
stripped true
subsys   Windows CUI
va       true
```
