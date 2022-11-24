---
title: "Recognizing patterns in memory"
date: 2022-11-24T08:30:09-07:00
draft: false
---

Something I find frustrating is how hard it is to teach debugging skills. I think the biggest reason is because there are many things that can only be learned through experience. One area where I think this is particularly true is anything that requires pattern recognition. Our brains are great at recognizing patterns, but it often takes a large amount of practice to be able to identify useful patterns in data.

I can't instantly give you pattern recognition skills with a short blog post, but I can tell you about some of the patterns that I look for so you can start to train your brain to see these as well. Recognizing patterns in memory can be useful as it can give you a hint for things like memory corruption, which are often some of the hardest errors to debug from a postmortem analysis. Getting a rough idea of what type data is ovewriting other data in a process can tell you where to look next and can help narrow down where an issue might be, because the bug is usually in the code that wrote this data (or in closely related code).

# Aligned 32/64-bit data

A frequent pattern in both file formats and data structures is aligned 32-bit or 64-bit data. Often we choose to use 32-bits for values that will never come close to filling the full space, simply because they are often efficient to use and are the default choice in cases where memory or storage isn't particularly constrained. Additionally, even when smaller data types are used, it's common for these values to be aligned to 32-bit or 64-bit boundaries. In both cases, we see distinct patterns.

Take this memory dump for example:

```
09 00 00 00 30 43 00 00 0A 55 02 00 10 00 00 00
E0 49 01 00 2A 0B 01 00 06 00 00 00 A8 00 00 00
78 06 00 00 07 00 00 00 38 00 00 00 EC 00 00 00
0F 00 00 00 54 05 00 00 24 01 00 00 0C 00 00 00
F8 3A 00 00 32 D0 00 00 15 00 00 00 EC 01 00 00
A4 2A 00 00 16 00 00 00 98 00 00 00 90 2C 00 00
```

When viewing the data as 8 bytes or 16 bytes per line, columns show up of all zero or all nonzero data. Column 0 has all nonzero data, column 3 has all zero data, and column 2 is mostly nonzero data. Looking at column 4, you can see again that it is all nonzero data. We can say with high confidence that there are a bunch of 32-bit values here.

## Pointers

A special case of aligned data is pointers. These are often very easy to spot. While a 64-bit pointer could span the entire address range, in reality you see a number of patterns in the addresses that we see in a Windows process.

Take this stack dump for instance:

```
0:000> dp @rsp
000000d3`f213ef00  00000000`00000000 000000d3`f213ef90
000000d3`f213ef10  00000000`00000000 000000d3`f213ef90
000000d3`f213ef20  00000000`00000000 00000000`00000040
000000d3`f213ef30  00000000`00000010 00007ffc`690e3ff5
000000d3`f213ef40  00007ffc`69135a00 00007ffc`69135900
000000d3`f213ef50  00007ffc`69135900 00007ffc`69135900
000000d3`f213ef60  000000d3`f213f034 00007ffc`00000004
000000d3`f213ef70  00000000`00000000 00000000`00000000
```

You can see two patterns that stand out. There are a few 64-bit values that start with ```00007ffc```, and a few that start with ```000000d3```. The ones that start with ```d3``` are stack addresses, and you can see that from the fact that RSP starts with ```d3``` as well. (cross-thread stack references are rare, but stacks will often start with the same 32-bits). The ```00007fXX``` sequence is also very common, and is usually loaded modules. The stack and heap addresses are somewhat random, but unless an application has high memory consumption, there will probably be only a few of these that are valid within a process. You can see this by running ```!address``` in a process with no parameters.

The nice part is that if something "looks like" an address, it's very easy to check if it actually is a valid address. Just run ```!address <address>```, which will tell you if the address is valid in the current process. The odds of a 64-bit value being a pointer by chance is quite low.

Other useful commands to use with an arbitrary address include ```ln``` to find the closest symbol (for addresess inside a module) and ```!heap``` to find information about a heap allocation.

# UTF-16 characters

A common text encoding in Windows is UTF-16. This is usually very easy to recognize since most hex viewers will show you the ASCII representation of data, and UTF-16 encoding will usually look like ascii characters with "blank characters" every other byte, like this:

```
..C.:.\.P.r.o.g.
r.a.m. .F.i.l.e.
s.\.W.i.n.d.o.w.
```

Obviously UTF-16 could look much different if they were for localized strings in other languages, but it's still very common to see strings where the characters are in the range of 0-127, where the interpretation overlap with the ASCII interpretation. (As a side note, I'll mention that no matter how careful you are, you will always find someone who thinks you have used the incorrect terminology when it comes to text encoding) 

If you are viewing just hex bytes, it's nearly as easy to recognize:

```
00 00 43 00 3A 00 5C 00 50 00 72 00 6F 00 67 00
72 00 61 00 6D 00 20 00 46 00 69 00 6C 00 65 00
73 00 5C 00 57 00 69 00 6E 00 64 00 6F 00 77 00
```

Every other column will be full of zeros, and the other columns will generally be 20-7F.

# Executable data

Each architecture is a bit different in the way that instructions are encoded. Most of my experience is with x86 and x64 code. It tends to be very easy to spot "normal" code that was generated by a compiler. Hand coded assembly can sometimes be harder to spot.

There are some easy tricks to spotting x64 code quickly. Take this data for instance:

```
F0 01 00 00 5B C3 CC CC CC CC CC CC CC CC CC CC
48 83 EC 38 48 83 64 24 20 00 41 B9 01 00 00 00
4C 8D 44 24 40 41 8D 51 10 48 C7 C1 FE FF FF FF
E8 0B CC FC FF 85 C0 78 0A 80 7C 24 40 00 75 03
CC EB 00 48 83 C4 38 C3 CC CC CC CC CC CC CC CC
```

The first thing you'll notice is the lack of alignment. X86 uses a variable length encoding, and there is generally not a reason to align code or data on 32-bit or 64-bit boundaries inside a function (although there is a reason to have 32-bit aligned addresses as the *start* of a function or other jump target). The second thing you will probably notice is the abundance of ```CC``` bytes. This is the one byte encoding for the ```int 3``` instruction, which is the instruction used to trigger a software breakpoint. These are sometimes inserted in between functions to align the entry address of the next function. Using ```CC``` bytes means a misjump to these locations will trigger an immediate breakpoint exception instead of accidentally running some arbitrary part of code. So you'll see these sequences of ```CC``` bytes, and they will always end at some aligned address. The other thing to notice about these sequences is that they usually start with a ```C3``` byte, which is single byte encoding for a ```ret``` instruction, which is very frequently (but not always) the last instruction of a function.

These tricks work very well for binaries that are part of Windows, as the compiler options used means these sequences will nearly always show up. Other compilers and compiler options won't have these. Or you may be looking at a small part of a larger function. For this, there are some other tricks I use. Take this code for instance:

```
41 54 55 57 56 53 48 83 EC 20 48 8B 35 CF 73 1F
00 85 D2 48 89 CF 89 D3 4C 89 C5 89 16 75 54 8B
05 5B 1D 21 00 85 C0 74 33 E8 A2 52 17 00 49 89
E8 31 D2 48 89 F9 E8 75 63 17 00 49 89 E8 89 DA
```

Two patterns I'll direct your attention to. The first is the sequence ```54 55 57 56 53```. Seems random, but these are all single byte "push" instructions. In fact, anything starting with a 5 is either a push or a pop instruction. These often show up in a sequence because push and pop instructions frequently show up clustered at the start or end of a function. The other thing to notice is that there are a bunch of ```4X``` bytes in this sequence. These are very common in x64 code because they are a prefix byte used to access the larger register set available in x64 compared to x86 (called the REX prefix). It's possible to see these in the middle of an instruction, of course, but more often than not this will mark the beginning of an instruction. If you see a ```4X``` byte followed by an ```8X``` byte, it's very likely a basic ALU operation or ```MOV``` operation. These represent a very large percent of most code. Just look through disassembly for any function and you will likely see a lot of ```8X``` or ```4X 8X``` sequences at the start of instructions.

You can always use a disassembler to test this theory and see if the instructions "look" right, but determining what looks right is easily the topic for another post.

# High entropy data

If you have something that doesn't show any obvious signs of alignment, and doesn't have any executable code, it's possible that you're looking at some high entropy data. Take this sequence:

```
02 4D 1C 07 18 CF 3F 2B F4 B0 50 F8 D9 6E 1D 91
83 4E DF 75 D0 BE B2 D2 9F AC 79 18 C4 67 A5 A0
47 04 EF F4 85 A0 53 8C 52 90 84 4C B7 22 4B C5
```

Here we see no obvious alignment, no sequences of ```CC```, no occurrences of ```C3```, and no sequences of ```4X 8X```. High entropy data could be any number of things, but it almost always falls into one of two categories:

1. Compressed data
2. Encrypted data

Without more context, it's generally not possible to identify the format of high entropy data or even to determine if a sequence of bytes is compressed or encrypted. There are probably some cases where you can find some data structures embedded for compressed data (like data about video frames for instance), but given that the compressed portion will usually be larger than the fixed layout structures, it will usually take more work to find. The best bet here is to try to find a header for the data, and see if there is some signature that tells you more. But if there's just a small snippet of high entropy data corrupting some other memory, you may want to consider what sort of high entropy data the application could be dealing with (could it be decompressing data? transmitting encrypted data over a network?)

# Conclusion

The idea of recognizing a sequences of bytes and finding the meaning behind them has always felt very satisfying to me. Have you ever found this useful? Do you have any tricks of your own? Let me know on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim)!