---
title: "Debugger Lies: Stack Corruption"
date: 2022-08-21T09:57:31-07:00
draft: false
---

There are lots of reasons your debugger might be lying to you. Sometimes it's because information is lost when compiling due to optimizations. Sometimes the symbolic debug information isn't expressive enough. Other times it can be due to a bug in the debugger (although I hope that reason is rare). One frustrating case where the debugger sometimes "lies" to you is the stack walk. It's the single most important information to come out of a crash, so when the stack walk is wrong, it's probably going to make analysis difficult. There are lots of why a stack walk can go bad, but one frequent cause is corrupted stack memory.

# Stack corruption

A stack corruption can be caused by buffer overflows on local variables because the return address and data for the calling functions are stored in addresses *higher* than the addresses for locals. For instance, a function that does something like this can corrupt the stack because it is writing data that is larger than what is allocated on the stack, and the return address is overwritten.

{{< highlight c >}}
int OverflowFunc()
{
    int buffer[80] = {};
    int* pbuf = buffer;
    memset(pbuf, 0, 500);
    return 1; 
}
{{< / highlight >}}

(Note that compilers are smart, so you sometimes need to jump through hoops to compile code that is obviously wrong like this. I added the extra ```pbuf``` variable to get this to compile and crash)

If you've compiled with "/GS" in MSVC, the top frame will be at __report_gsfailure, and the second frame will be at TestFunc, but it will stop abruptly with no further frames. 

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 0000008b`954ff6e0 00007ff7`79cd11c7     WindowsConsoleApp!__report_gsfailure+0x1d
01 0000008b`954ff720 00000000`00000000     WindowsConsoleApp!OverflowFunc+0x57
```

In this case, the return address for OverflowFunc and the [security cookie](https://docs.microsoft.com/en-us/archive/msdn-magazine/2017/december/c-visual-c-support-for-stack-based-buffer-protection) have been overwritten, triggering a GS failure. Because the crash is before the function returns, you can still see which function has a corrupted stack (which is usually the function to start looking at for bugs). The return address has been overwritten, so the debugger does not know what function called into OverflowFunc and the stack stops there, rather than telling you what function called into OverflowFunc.

If you do not compile with "/GS", you'll end up with a completely useless stack output.

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 000000f5`fb4ff770 00000000`00000000     0x0
```

It's nearly always worth enabling /GS, for both security and debugging reasons. The cost is small compared to the benefits.

If you need to debug a stack corruption, there are a few approaches I can suggest.

## Memory access breakpoints

If you can locally reproduce the issue and can break into the debugger at the start of the function that will have a corrupt return address (such as OverflowFunc in the above example), then you can find the instruction where the stack is corrupted by using a memory access breakpoint. To start, we can break in at the start of OverflowFunc. Then we want to find the current stack pointer, since it will be pointing to the return address. You can see this in the output of "k" in the "Child-SP" column, but you can also evaluate it directly using ```dx @$csp```

```
0:000> k2
 # Child-SP          RetAddr               Call Site
00 00000061`887cf898 00007ff7`3b5c1189     WindowsConsoleApp!OverflowFunc
01 00000061`887cf8a0 00007ff7`3b5c1a80     WindowsConsoleApp!main+0x9
0:000> dx @$csp
@$csp                 : 0x61887cf898 : 0x89 [Type: unsigned char *]
    0x89 [Type: unsigned char]
```

We can set a memory access breakpoint on this address using ```ba```:

```
0:000> ba w 8 @$csp
0:000> bl
     0 e Disable Clear  00000061`887cf898 w 8 0001 (0001)  0:**** 
```

Continuing execution now will break in at the exact instruction that corrupts the return address.

```
0:000> g
Breakpoint 0 hit
WindowsConsoleApp!OverflowFunc+0x2e:
00007ff7`3b5c116e f3aa            rep stos byte ptr [rdi]
0:000> lsa .
   102: int OverflowFunc()
   103: {
   104:     int buffer[80] = {};
   105:     int* pbuf = buffer;
>  106:     memset(pbuf, 0, 500);
   107:     return 1;
   108: }
```

This is an easy way to find the stack corruption in cases where you can get a repro under the debugger and can set a breakpoint prior to the stack corruption. If that's not possible, either because you can't get a local repro, or because you can't break into the debugger for the function invocation that is causing the stack corruption then the next option you should consider is using Time Travel Debugging.

## Using Time Travel Debugging

TTD ([Time Travel Debugging](https://aka.ms/ttd)) is incredibly useful for many types of bugs, but it is particularly well suited to debugging memory corruption such as a stack corruption. First, [record a trace](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-record) of the issue and open it in WinDbg. Run to the end of the trace with ```g``` and you'll see the crash like we did before:

```
(80f0.7570): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 63:1
WindowsConsoleApp!__report_gsfailure+0x1d:
00007ff7`99a81bf5 cd29            int     29h
0:000> k
 # Child-SP          RetAddr               Call Site
00 00000067`064ff760 00007ff7`99a811c7     WindowsConsoleApp!__report_gsfailure+0x1d
01 00000067`064ff7a0 00000000`00000000     WindowsConsoleApp!OverflowFunc+0x57
```

Since this is a TTD trace, we can go back to the beginning of the function execution and see when the return address is overwritten. Set a breakpoint on the beginning of the function and execute backwards with ```g-```. (You may need to execute a ```t-``` first if you get "stuck" and the debugger breaks in again at the same instruction)

```
0:000> bp WindowsConsoleApp!OverflowFunc
0:000> t-
Time Travel Position: 63:0
WindowsConsoleApp!__report_gsfailure+0x1d:
00007ff7`99a81bf5 cd29            int     29h
0:000> g-
Breakpoint 0 hit
Time Travel Position: 62:28
WindowsConsoleApp!OverflowFunc:
00007ff7`99a81170 4057            push    rdi
```

Since we know the return address is corrupted, we can set a memory access breakpoint on the return address on the stack using ```ba```. And since we are at the beginning of the function, the return address will just be at the stack pointer:

```
0:000> dqs @rsp L1
00000067`064ff908  00007ff7`99a811d9 WindowsConsoleApp!main+0x9
0:000> ba w 8 @rsp
0:000> g
Breakpoint 0 hit
Time Travel Position: 62:2CF
WindowsConsoleApp!OverflowFunc+0x40:
00007ff7`99a811b0 f3aa            rep stos byte ptr [rdi]
```

The ```ba w 8 @rsp``` command sets a write breakpoint of 8 bytes on the memory that RSP is pointing to. When we continue execution again, we'll break in at the exact instruction that overwrites that memory address. This example is very simple and contrived, but this technique works well even for very complicated cases.

TTD is an incredible tool, but there are some cases where it can't be used. The technology behind TTD is using CPU emulation, and the performance cost of the CPU emulation might make it impossible to use for some applications. For cases where you can't use TTD to find your bug, there are some other options for you to consider.

## Using the shadow stack

If you are lucky enough to be debugging a crash on a new CPU that supports [hardware-enforced stack protection](https://techcommunity.microsoft.com/t5/windows-kernel-internals-blog/developer-guidance-for-hardware-enforced-stack-protection/ba-p/2163340), such as Intel's Control-flow Enforcement Technology (CET), you have another option for analysis using the shadow stack. This is an independent stack that tracks return addresses and can't be overwritten directly, so it's safe from buffer overflow. While this hardware feature wasn't primarily intended for debugging, it turns out that it's *incredibly* useful as one. You can dump the symbols of the shadow stack using ```dps @ssp```. This will show you the call chain even if the main stack is completely corrupted.

## Manual stack recovery

If only part of the stack is corrupt, it may be possible to manually recover the healthy part of the stack. The main command to be using is ```dps @rsp```. After that, you can repeatedly run a ```dps``` command with no parameters to continue dumping symbols at higher addresses. You will likely start to see function names from the undamaged part of the stack. When doing this you may see that there are other symbol names sitting on the stack from unrelated call chains that by chance have not yet been overwritten. So you need to be careful and make sure that the stack makes sense for the crash you are looking at. Even with the potential for wrong information, it's a useful tool that can give you more information on how the crash happened.

If you want to get the full undamaged portion of the stack once you find a valid frame, you can dump a stack from an arbitrary starting point by changing the registers with the ```r``` command and then using the ```k``` command, or by giving the ```k``` command an explicit context to unwind from using the ```k = <StackPtr> <InstructionPtr> <FrameCount>``` syntax. (Documentation on that is [here](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/k--kb--kc--kd--kp--kp--kv--display-stack-backtrace-)).

Raymond Chen has gone into more depth on this in a blog post called [How to rescue a broken stack trace on x64: Recovering the stack pointer](https://devblogs.microsoft.com/oldnewthing/20130906-00/?p=3303).

## AddressSanitizer

If you can reproduce the problem locally and have the ability to recompile the source, there are a number of tools that can instrument your code to find stack corruptions. One that is built into MSVC is [AddressSanitizer](https://docs.microsoft.com/en-us/cpp/sanitizers/asan) (also called ASan). Enabling this is as easy as adding one extra compiler option: ```/fsanitize=address```. Running the example above and we now get a crash with some additional diagnostics:

```
    #1 0x7ff6f8811677 in OverflowFunc C:\git\book\ExampleCode\WindowsConsoleApp\WindowsConsoleApp.cpp:106
    #2 0x7ff6f88116c8 in main C:\git\book\ExampleCode\WindowsConsoleApp\WindowsConsoleApp.cpp:112
    #3 0x7ff6f8812e6f in __scrt_common_main_seh D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl:288
    #4 0x7ffc72867033 in BaseThreadInitThunk+0x13 (C:\WINDOWS\System32\KERNEL32.DLL+0x180017033)
    #5 0x7ffc73a42650 in RtlUserThreadStart+0x20 (C:\WINDOWS\SYSTEM32\ntdll.dll+0x180052650)

Address 0x0009f930f7a0 is located in stack of thread T0 at offset 352 in frame
    #0 0x7ff6f881158f in OverflowFunc C:\git\book\ExampleCode\WindowsConsoleApp\WindowsConsoleApp.cpp:103

  This frame has 1 object(s):
    [32, 352) 'buffer' <== Memory access at offset 352 overflows this variable
HINT: this may be a false positive if your program uses some custom stack unwind mechanism, swapcontext or vfork
      (longjmp, SEH and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow (C:\git\book\ExampleCode\WindowsConsoleApp\x64\Release\clang_rt.asan_dynamic-x86_64.dll+0x18004b04d) in _asan_wrap_GlobalSize+0x481b7
Shadow bytes around the buggy address:
  0x012d53661ea0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x012d53661eb0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x012d53661ec0: 00 00 00 00 00 00 00 00 f1 f1 f1 f1 00 00 00 00
  0x012d53661ed0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x012d53661ee0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x012d53661ef0: 00 00 00 00[f3]f3 f3 f3 00 00 00 00 00 00 00 00
  0x012d53661f00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
...
```

The diagnostic output will usually be enough to pinpoint the source of the corruption. If not, you can still navigate to the stack frames for the blamed function and see what the state of the function is when the corruption occurs. Since AddressSanitizer works by replacing the call to memset (as well as every other memory access) with an instrumented version that checks the bounds, the generated code will not match the original executable, but in most cases it should be equivalent.

## Debugger scripting

If for some reason you:

* Can't break into a function prior to stack corruption
* Can't use TTD
* Can't recompile with AddresSanitizer
* Can't use hardware shadow stack
* Aren't able to debug the issue by manually reconstructing the stack

Then I have one other option for you to consider. You can use a debugger script to "instrument" a function looking for any memory access that would overwrite the return address of a function. We can do this using javascript and the [conditional breakpoint](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-a-conditional-breakpoint) feature. What we want to do is to set a breakpoint at the start of the function that will create a memory access breakpoint on the stack value that contains the return address. Then set a second breakpoint that will clear the first breakpoint so that we stop watching the stack location if no stack corruption occurs. We can do that with a debugger JS script like this:

{{< highlight js >}}
"use strict";

function initializeScript()
{
    return [new host.apiVersionSupport(1, 7)];
}

function onFunctionEntry()
{
    var stackPtr = host.namespace.Debugger.State.PseudoRegisters.General.csp;
    var bp = host.namespace.Debugger.Utility.Control.SetBreakpointForReadWrite(stackPtr, "w", 8);
    var id = bp.Id;
    var returnAddr = host.namespace.Debugger.State.PseudoRegisters.General.ra;
    var addrString = returnAddr.address.toString(16);
    host.namespace.Debugger.Utility.Control.ExecuteCommand("bp /1 " + addrString + '"bc ' + id + '; g"');
    return false;
}
{{< / highlight >}}

To use this script, we need to set a breakpoint at the start of the function to examine using ```/w``` to execute the JS script.

```
0:000> bp /w "@$scriptContents.onFunctionEntry()" OverflowFunc
0:000> g
Breakpoint 1 hit
WindowsConsoleApp!OverflowFunc+0x2e:
00007ff7`3b5c116e f3aa            rep stos byte ptr [rdi]
0:000> lsa .
   102: int OverflowFunc()
   103: {
   104:     int buffer[80] = {};
   105:     int* pbuf = buffer;
>  106:     memset(pbuf, 0, 500);
```

A more complete version of this script is on the [WinDbgCookbook repo](https://github.com/TimMisiak/WinDbgCookbook).

# Conclusion

I don't intend for this post to be exhaustive, but I think I've covered a wide range of approaches for debugging stack corruption. I'm sure there are many more approaches, so let me know if there is a technique that you use that I didn't mention.