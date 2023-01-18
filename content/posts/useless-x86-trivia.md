---
title: "Weird things I learned while writing an x86 emulator"
date: 2023-01-09T23:54:20-08:00
draft: true
---

If you've read my [first post about assembly language](/posts/fakers-guide-to-assembly), you might have thought I'd write some more about how to understand assembly language. I will, but this post is not that. This post is going to talk about some of the weird things I learned while writing an x86 and amd64 emulator. The emulator I wrote was for [Time Travel Debugging](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview). One piece of the TTD technology is a CPU emulator, which is used to record the entire execution of a process at an instruction level. The first version of TTD was called iDNA, and the emulator for iDNA was written almost completely in assembly code, which was very fast for the time, but was completely unmaintainable as you can imagine.

I wasn't involved in the first version, but I was involved in the second version, where we rewrote the emulation portion (and eventually most other parts) of iDNA. The new one was written in C++, and we aimed to achieve most of the performance of the assembly language version while having a more maintainable code base. Writing a CPU emulator is, in my opinion, the absolute best way to REALLY understand how a CPU works. You need to pay attention to every detail. So let me tell you about some of the details I learned, starting with some things that are a bit less obscure.

# Useless x86 encoding trivia

The x86 encoding scheme is a bit funny in that there are often multiple ways to encode the exact same instruction. The ```int 3``` instruction can be encoded as ```CD 03```, but can also be encoded in a single byte of ```CC```. This is a very useful encoding because ```int 3``` is used as a software breakpoint. That way, it's always possible to set a breakpoint at any point in a function, even if it's an instruction that lands at the end of a memory page (with no page mapped after it). Many of the encodings are just designed to be shorter for some common cases. Adding an immediate value to EAX or RAX with the ```ADD``` instruction can be expressed in a compact form. For instance:

```
05cccccccc      add     eax,0CCCCCCCCh
```

Doing the same for ECX takes an extra byte:

```
81c1cccccccc    add     ecx,0CCCCCCCCh
```

So while both registers are general purpose, the fact that "EAX" is called the "Accumulator register" is not just a convention, it actually makes a difference to the encoding (and potentially the performance, as a result).

Equivalent instructions are more than just multiple encodings. Instructions in x86 can take "prefix bytes" which modify the behavior of the instruction. The "REX" set of prefixes are commonly used in 64 bit code to access a wider range of registers compared to 32-bit code (and make code sequences [easier to recognize](/posts/recognizing-patterns)). An x86 CPU will happily take one of these prefixes, even if it doesn't have any effect. Put a "REX" byte on an 8-bit add, and it does nothing:

```
4004cc          add     al,0CCh
```

In fact, you can put TWO of these prefixes on. Many disassemblers (including the one in WinDbg) will get confused, but the CPU will execute it just fine:

```
0:000> rrax=0
0:000> u . L2
00007ff8`22160950 40              ???
00007ff8`22160951 4004cc          add     al,0CCh
0:000> t
00007ff8`22160954 cc              int     3
0:000> rrax
rax=00000000000000cc
```

Can we add as many prefixes as we want? Almost. Until you get to 15 bytes. This is a hard limit on current x86 compatible CPUs. Any instruction longer than 15 bytes is considered invalid. Some prefixes are also strict about when they can be used, such as the ```LOCK``` prefix (F0).

There are also some other weird prefixes that you might not know about. The "Address override" prefix can be used to reference a 32-bit address when running in 64-bit mode.

```
488d0424        lea     rax,[rsp]
67488d0424      lea     rax,[esp]
```

And when running 32-bit code, it will switch the address mode to 16-bit addresses! I don't know that I've ever seen a compiler generate code like that, but it's there. Something interesting to note, however, is that disassembling these bytes relies on knowing the default operand size and address size of a code segment (which can be [configured by the kernel](https://en.wikipedia.org/wiki/Segment_descriptor)). If we disassemble some bytes in 32-bit mode and 64-bit mode, we can get different results.

In 32-bit mode:

```
8b0424          mov     eax,dword ptr [esp]
```

In 64-bit mode:
```
8b0424          mov     eax,dword ptr [rsp]
```

It's not just address sizes either. The entire range that was previous dedicated to ```INC reg``` and ```DEC reg``` on x86 (40-4F) is now used for the REX prefix bytes on x64!

In 32-bit mode, we have this:

```
48              dec     eax
030424          add     eax,dword ptr [esp]
```

But in 64-bit mode, those two instructions have merged to form a single instruction! 
```
48030424        add     rax,qword ptr [rsp]
```

The encoding space used by ```INC``` and ```DEC```, so it's understandable why the AMD designers decided to use these bytes for the new prefixes that would expand the register set in 64 bit mode. There was already a different encoding for these instructions that supported both registers and memory locations, so nothing was lost.

# Weird instruction behavior

Speaking of ```INC``` and ```DEC```, there's a slightly odd aspect of these instructions that's worth noting. You might thing that ```INC EAX``` does the same thing as ```ADD EAX, 1```, but they are slightly different because ```ADD``` will update the carry flag but the ```INC``` instruction does not modify it! This is an easy thing to miss, and when writing the TTD emulator I got this wrong. 