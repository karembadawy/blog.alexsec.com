---
layout: post
title: "BSidesIndy CTF 2017 library_book"
date: 2017-3-11 00:00:00
description: Writeup for BSidesIndy2017 library_book challenge using radare2
tags:
 - radare2
 - reverse engineering
---

>library_book
>Note: signal 11 doesn't mean broken.
>[library_book](/files/bsidesindy2017/library_book)

It was embaracing that we were the only team who solved this challenge! Even some formidable teams didn't solve this one.
First step is always trying to run the challange and see what we get
```
➜ file library_book 
library_book: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, stripped
➜ chmod +x library_book
➜ ./library_book 
[1]    26662 segmentation fault (core dumped)  ./library_book
➜  
```
So it was really meant to give a segmentation fault... the only hope we got is mainly static analysis.
``` bash
➜r2 library_book 
[f000:fff0]> i
type     bios
file     library_book
fd       6
size     0x4f97c
iorw     false
blksz    0x0
mode     -r--
block    0x100
format   bios
havecode true
pic      false
canary   false
nx       false
crypto   false
va       true
bintype  bios
class    1.0
arch     x86
bits     16
machine  pc
os       any
minopsz  1
maxopsz  16
pcalign  0
subsys   unknown
endian   little
stripped false
static   true
linenum  false
lsyms    false
relocs   false
binsz    326012

[f000:fff0]> 
```
Now things started to get more confusing. It will make sense why it segfaulted if it was bios. After a quick look at the strings using `izz` command, I found these strings which means that indeed it is not a bios file
```
$Info: This file is packed with the BSI executable packer http://BSI.sf.net $\n
$Id: BSI 3.91 Copyright (C) 1996-2013 the BSI Team. All Rights Reserved. $\n
GCC: (Ubuntu 5.4.0-6u\e1~16
```
After checking the website, I realized that BSI is not a binary packer at all! Actually the first string looks extremly similar to the one made by UPX packer, So I would try replacing all instances of `BSI` with `UPX` then unpacking it using UPX
```
[f000:fff0]> e search.from=0x0
[f000:fff0]> e search.to = 0xffffffff
[f000:fff0]> / BSI
Searching 3 bytes from 0x00000000 to 0xffffffff: 42 53 49 
Searching 3 bytes in [0x0-0xffffffff]
hits: 13
0x000000b4 hit0_0 . $(jBSI! .
0x0004ecb2 hit0_1 .packed with the BSI executable pack.
0x0004eccf hit0_2 .e packer http://BSI.sf.net $$Id: .
0x0004ece2 hit0_3 ..sf.net $$Id: BSI 3.91 Copyright .
0x0004ed07 hit0_4 .) 1996-2013 the BSI Team. All Right.
0x0004f950 hit0_5 .Y_$IBSI!BSI!.
0x0004f958 hit0_6 .$IBSI!BSI!_h.
0x000ff336 hit0_7 .packed with the BSI executable pack.
0x000ff353 hit0_8 .e packer http://BSI.sf.net $$Id: .
0x000ff366 hit0_9 ..sf.net $$Id: BSI 3.91 Copyright .
0x000ff38b hit0_10 .) 1996-2013 the BSI Team. All Right.
0x000fffd4 hit0_11 .Y_$IBSI!BSI!.
0x000fffdc hit0_12 .$IBSI!BSI!_h.
[f000:fff0]> oo+
File library_book reopened in read-write mode
[f000:fff0]> s 0xb4
[0000:00b4]> w UPX
[0000:00b4]> s 0x0004ecb2
[4000:ecb2]> w UPX
[4000:ecb2]> s 0x0004eccf
[4000:eccf]> w UPX
[4000:eccf]> s 0x0004ece2
[4000:ece2]> w UPX
[4000:ece2]> s 0x0004ed07
[4000:ed07]> w UPX
[4000:ed07]> s 0x0004f950
[4000:f950]> w UPX
[4000:f950]> s 0x0004f958
[4000:f958]> w UPX
[4000:f958]> s 0x000ff336
[f000:f336]> w UPX
[f000:f336]> s 0x000ff353
[f000:f353]> w UPX
[f000:f353]> s 0x000ff366
[f000:f366]> w UPX
[f000:f366]> s 0x000ff38b
[f000:f38b]> w UPX
[f000:f38b]> s 0x000fffd4
[f000:ffd4]> w UPX
[f000:ffd4]> s 0x000fffdc
[f000:ffdc]> w UPX
[f000:ffdc]> q
➜ upx -d library_book
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2013
UPX 3.91        Markus Oberhumer, Laszlo Molnar & John Reiser   Sep 30th 2013

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    838610 <-    326012   38.88%  linux/ElfAMD   library_book

Unpacked 1 file.
➜ 
```
The command `oo+` is used to enable writing in the binary file, thus patching it. `w` is used to write string at the current seek, and `s` is used to change the current seek.
The `-d` switch in `upx` command is used to unpack binaries packed with upx. The good news is that upx unpacked the binary file successfually and it recognized it as ELF file for AMD processors, The only problem is that the executable still give signal 11.
```
➜ ./library_book
[1]    16790 segmentation fault (core dumped)  ./library_book
➜ file library_book 
library_book: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.32, BuildID[sha1]=32e423425aca449a1b5e60722880c9128042f935, stripped
➜ r2 library_book
Warning: Cannot initialize dynamic strings
[0x00400890]> i
type     EXEC (Executable file)
file     library_book
fd       6
size     0xcc418
iorw     false
blksz    0x0
mode     -r--
block    0x100
format   elf64
havecode true
pic      false
canary   false
nx       true
crypto   false
va       true
bintype  elf
class    ELF64
lang     c
arch     x86
bits     64
machine  AMD x86-64 architecture
os       linux
minopsz  1
maxopsz  16
pcalign  0
subsys   linux
endian   little
stripped true
static   true
linenum  false
lsyms    false
relocs   false
rpath    NONE
binsz    834645
[0x00400890]>
```
So The next step is to auto analyze the whole code (That would take long time) and try some FLIRT signatures against the binary hoping that one will work. I got mine from [push0ebp](https://github.com/push0ebp/sig-database)'s repo on github, and I tried all of the ubuntu's ones all together using the `zF` command in r2!
After that I would go and check the function main to see what is in there
```
          ┌────────────────────┐
          │ [0x4009ae] ;[ga]   │
          └────────────────────┘
                     v
                     │
                     │
                     .───────────────────────────────────────────────────.
            ┌────────────────────┐                                       │
            │  0x4009bd ;[gc]    │                                       │
            └────────────────────┘                                       │
                     t   f                                               │
            ┌────────┘   └──────────────────────────┐                    │
            │                                       │                    │
            │                                       │                    │
┌───────────────────────────────┐     ┌───────────────────────────────┐  │
│  0x4009fb ;[gb]               │     │  0x4009c3 ;[ge]               │  │
│ 0x00400a00 call flirt.putchar │     │ 0x004009f0 call flirt.putchar │  │
└───────────────────────────────┘     └───────────────────────────────┘  │ 
                                                    └────────────────────┘

```
So it seams that The main function as simple as calling `puchar` in a loop, then calling `putchar` one last more time. What makes us confident that flirt guessed putchar correctly is that the second `putchar` looks like this:
{% highlight nasm linenos %}
0x004009fb      bf0a000000     mov edi, 0xa
0x00400a00      e8cbf00000     call flirt.putchar          ;[2]; int putchar(int c)
0x00400a05      b800000000     mov eax, 0
0x00400a0a      c9             leave
{% endhighlight nasm %}
if we assumed that this putchar doesn't use standard linux calling convention AKA cdecl, and argument is put into register `edi` then this `putchar` print newline, and that make sense here to print new line after finishing.
The loop part do some calculations that I am lazy to follow up with specially when it comes to sign extensions and all that headache
{% highlight nasm linenos %}
0x004009bd     837dfc18      cmp dword [rbp - local_4h], 0x18 ; [0x18:4]=0x400890 entry0
0x004009c1     7f38          jg 0x4009fb                 ;[1]
0x004009c3     8b45fc        mov eax, dword [rbp - local_4h]
0x004009c6     4898          cdqe
0x004009c8     8b0485a0906c  mov eax, dword [rax*4 + 0x6c90a0] ; [0x6c90a0:4]=0x2303526
0x004009cf     25ffffff00    and eax, 0xffffff
0x004009d4     89c2          mov edx, eax
0x004009d6     8b45fc        mov eax, dword [rbp - local_4h]
0x004009d9     4898          cdqe
0x004009db     8b0485a0906c  mov eax, dword [rax*4 + 0x6c90a0] ; [0x6c90a0:4]=0x2303526
0x004009e2     c1e818        shr eax, 0x18
0x004009e5     89c1          mov ecx, eax
0x004009e7     d3fa          sar edx, cl
0x004009e9     89d0          mov eax, edx
0x004009eb     0fb6c0        movzx eax, al
0x004009ee     89c7          mov edi, eax
0x004009f0     e8dbf00000    call flirt.putchar          ;[2]; int putchar(int c)
0x004009f5     8345fc01      add dword [rbp - local_4h], 1
0x004009f9     ebc2          jmp 0x4009bd                ;[3]
; JMP XREF from 0x004009c1 (main)
{% endhighlight nasm %}
Like the other call to `putchar`, argument seems to be passed via `edi`.
Here is the strategy to solve this challenge.
We overwrite both calls to `putchar` with nops so no function get called thus avoid any trouble with that might lead to segmentation fault, from entry0 go directly to main function, set break point at `0x004009f0` and see what goes in edi.
{% highlight python linenos %}
import r2pipe
dbg = r2pipe.open("library_book", ["-d"])
dbg.cmd("dr rip=0x004009ae")
print "[+] rip is set to", dbg.cmd("dr rip")
dbg.cmd("db 0x004009ee")
flag = ""
for _ in xrange(0x19):
	dbg.cmd("dc")
	c = chr(int(dbg.cmd("dr edi"), 16))
	flag = flag + c
print flag
{% endhighlight python %}
