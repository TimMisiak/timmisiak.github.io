---
title: "Writing a Debugger From Scratch - DbgRs Part 1 - Attaching to a Process"
date: 2023-02-13T9:20:24-07:00
draft: false
---

I've left the Microsoft Debugger Platform team twice, and each time I've started writing my own debugger. I must really like debuggers or something. This time, I have two reasons for writing a new debugger. The first is because I want to learn Rust better, and writing something I already understand pretty well seems like a good way to learn. The second reason is to make it easier for people to learn how a debugger works. Using Rust also helps here because there are lots of crates that can take care of things like symbols and disassembly, and it will let us focus on the core ideas involved in writing a debugger.

I'm a Rust novice, and if you're looking to learn Rust better, I'd suggest to start with [https://www.rust-lang.org/learn](https://www.rust-lang.org/learn). I'm not going to try to explain too much about the Rust code, but the main concepts and APIs will apply across language. You should be able to follow along even without much knowledge of Rust. Those of you who read this and know Rust better than me, please feel free to [create issues](https://github.com/TimMisiak/dbgrs/issues) in the GitHub repo. Or even better, make a pull request! The code for this first part is in a branch called "part1", and you can see the full code [here](https://github.com/TimMisiak/dbgrs/tree/part1). If you want to follow along, I suggest cloning the whole repo locally and viewing it in a good IDE like VS Code with the rust-analyzer plugin.

# What is a debugger?

First, what is a debugger? There are lots of tools that are "debugging tools" but when most people hear "debugger" they usually think of a tool that can analyze either a running system or a static snapshot (such as a core file, crash dump, or VM snapshot)<sup>1</sup>. Most debuggers like GDB, LLDB, Visual Studio, and WinDbg can do both of these things. We'll start by creating a debugger that supports live usermode debugging on Windows. The core concepts will apply to nearly any OS, with the main differences being the OS APIs for debugging a process and the terminology that is used for certain concepts.

In Windows, there are just a few APIs that provide the necessary functionality for implementing a debugger. Most of these are described on MSDN in the [Debugging Functions](https://learn.microsoft.com/en-us/windows/win32/debug/debugging-functions) page. Note that these are the APIs that are used *by* a debugger such as Visual Studio or WinDbg. The core engine of WinDbg is called DbgEng, and has its own set of APIs that can be accessed through the [IDebugClient interface and related APIs](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nn-dbgeng-idebugclient). These higher level APIs and are very powerful, with built in stack unwinding, symbol analysis, flow control, disassembly, scripting, and lots of other capabilities. If you want to automate some debugging task, this is a better API to use. But that's not what we're going to do in this post series. We are going to implement a debugger *from scratch* using the most basic APIs provided by the OS.

There are a number of other functions we'll need to interact with a running process, but these APIs are not debugging-specific. We'll cover those as we encounter them.

# Basic structure

At the center of a live debugging session is an event loop. A debugger registers for debug events from a target process by "attaching" to it. For each event that occurs, the OS will freeze the target process and notify the debugger with information about that debug event. The debugger then has the opportunity to examine or manipulate the state of the target process. The debugger then continues from the debug event.

There's a ton of complexity embedded in here, but most of that complexity is in examining and manipulating the state of the target process. We'll start by creating a debugger that attaches to a process, monitors the debug events, and continues from them to allow the process to execute normally. That part is relatively straightforward, and will give us a basis for other functionality that can be built on top of this.

# Attaching to a process

On Windows, there are two main ways to attach to a process. If there is already a process running that we want to attach to, we can use the [DebugActiveProcess](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-debugactiveprocess) API, which takes a single parameter with the process ID that we want to debug.<sup>2</sup>. There is no handle or any other information passed back as a result of this API call. Instead, an implicit connection is created between the target process and the debugger process. To stop debugging a process, you call [DebugActiveProcessStop](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-debugactiveprocessstop), which also takes a single parameter of a process ID.

The second way to attach to a process is by creating the process in an attached state. To do that, we use the same [CreateProcessW](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) function that you would normally use to debug a process and include the ```DEBUG_ONLY_THIS_PROCESS``` or ```DEBUG_PROCESS``` flag as part of the ```dwCreationFlags``` parameter (documented as part of the [Process Creation Flags](https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags)). The difference between these flags is whether you want to debug only the specific process you launched or if you want to debug that process and all of its child processes.

We'll use CreateProcessW to start debugging a process:

```rust {linenos=true,lineNumbersInTable=false,linenostart=105}
    let mut si: STARTUPINFOEXW = unsafe { std::mem::zeroed() };
    si.StartupInfo.cb = std::mem::size_of::<STARTUPINFOEXW>() as u32;
    let mut pi: PROCESS_INFORMATION = unsafe { std::mem::zeroed() };
    let ret = unsafe {
        CreateProcessW(
            null(),                                       // lpApplicationName
            command_line_buffer.as_mut_ptr(),             // lpCommandLine
            null(),                                       // lpProcessAttributes
            null(),                                       // lpThreadAttributes
            FALSE,                                        // bInheritHandles
            DEBUG_ONLY_THIS_PROCESS | CREATE_NEW_CONSOLE, // dwCreationFlags
            null(),                                       // lpEnvironment
            null(),                                       // lpCurrentDirectory
            &mut si.StartupInfo,                          // lpStartupInfo
            &mut pi,                                      // lpProcessInformation
        )
    };
```
<small>[main.rs line 114](https://github.com/TimMisiak/dbgrs/blob/part1/src/main.rs#L105)</small>

There are a lot of parameters to CreateProcessW, but we can ignore the optional parameters to start. Later, we may want to give a user of this debugger the ability to customize certain aspects, but for now we'll simply use the entire command line passed in as the lpCommandLine. If you want to see how we got the command line to launch, see [parse_command_line in the github repo](https://github.com/TimMisiak/dbgrs/blob/part1/src/main.rs#L29) for this project. We'll also pass ```DEBUG_ONLY_THIS_PROCESS``` flag as part of dwCreationFlags. I've also specified ```CREATE_NEW_CONSOLE``` so that we won't share a console when we debug a console app (CDB/NTSD will share a console with the target and I always found that confusing, and rarely useful). Finally, we'll receive some [important information](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/ns-processthreadsapi-process_information) as part of the lpProcessInformation parameter, which is an output parameter.

When CreateProcessW returns, the process will be initially suspended. No code will execute until the debugger process starts handling the debug events. So that's what we'll do next. 

# Monitoring debug events

Two functions are used to drive the event loop for a debugger in Windows. The first is the [WaitForDebugEventEx](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-waitfordebugeventex) function<sup>3</sup>. The second function is [ContinueDebugEvent](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-continuedebugevent), which will resume the target process after handling the event. We will call these functions from [main_debugger_loop](https://github.com/TimMisiak/dbgrs/blob/part1/src/main.rs#L61).

```rust {linenos=true,lineNumbersInTable=false,linenostart=56}
    loop {
        let mut debug_event: DEBUG_EVENT = unsafe { std::mem::zeroed() };
        unsafe {
            WaitForDebugEventEx(&mut debug_event, INFINITE);
        }
```
<small>[main.rs line 56](https://github.com/TimMisiak/dbgrs/blob/part1/src/main.rs#L56)</small>

Something interesting you might note here is that the ```WaitForDebugEventEx``` function does not take in any process identifier. If a process is attached to multiple target processes, any call into WaitForDebugEventEx will monitor events for *all* attached processes<sup>4</sup>. The ```INFINITE``` parameter means that we will wait forever until a debug event is received. The event is described as a [DEBUG_EVENT](https://learn.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-debug_event) structure, which has fields for the thread ID, process ID, and an event code. Each event type has its own data structure, and the structures are stored as a union. You use the event code to determine which structure in the union is valid.

```cpp
typedef struct _DEBUG_EVENT {
  DWORD dwDebugEventCode;
  DWORD dwProcessId;
  DWORD dwThreadId;
  union {
    EXCEPTION_DEBUG_INFO      Exception;
    CREATE_THREAD_DEBUG_INFO  CreateThread;
    CREATE_PROCESS_DEBUG_INFO CreateProcessInfo;
    EXIT_THREAD_DEBUG_INFO    ExitThread;
    EXIT_PROCESS_DEBUG_INFO   ExitProcess;
    LOAD_DLL_DEBUG_INFO       LoadDll;
    UNLOAD_DLL_DEBUG_INFO     UnloadDll;
    OUTPUT_DEBUG_STRING_INFO  DebugString;
    RIP_INFO                  RipInfo;
  } u;
} DEBUG_EVENT, *LPDEBUG_EVENT;
```

We'll dig into that data in a later post. For now, we can just output which event we received.

```rust {linenos=true,lineNumbersInTable=false,linenostart=62}
        match debug_event.dwDebugEventCode {
            EXCEPTION_DEBUG_EVENT => println!("Exception"),
            CREATE_THREAD_DEBUG_EVENT => println!("CreateThread"),
            CREATE_PROCESS_DEBUG_EVENT => println!("CreateProcess"),
            EXIT_THREAD_DEBUG_EVENT => println!("ExitThread"),
            EXIT_PROCESS_DEBUG_EVENT => println!("ExitProcess"),
            LOAD_DLL_DEBUG_EVENT => println!("LoadDll"),
            UNLOAD_DLL_DEBUG_EVENT => println!("UnloadDll"),
            OUTPUT_DEBUG_STRING_EVENT => println!("OutputDebugString"),
            RIP_EVENT => println!("RipEvent"),
            _ => panic!("Unexpected debug event"),
        }
```
<small>[main.rs line 62](https://github.com/TimMisiak/dbgrs/blob/part1/src/main.rs#L62)</small>

We will loop forever handling debug events, but if we get an ```EXIT_PROCESS_DEBUG_EVENT```, we can just exit the loop because we are only debugging a single process.

```rust {linenos=true,lineNumbersInTable=false,linenostart=75}
        if debug_event.dwDebugEventCode == EXIT_PROCESS_DEBUG_EVENT {
            break;
        }
```
<small>[main.rs line 75](https://github.com/TimMisiak/dbgrs/blob/part1/src/main.rs#L75)</small>

When we are done handling the event, we'll call [ContinueDebugEvent](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-continuedebugevent), which will allow the target process to resume execution. This function takes three parameters: a process ID, a thread ID, and a "continue status". The first two are straightforward, and we can just use the process ID and thread ID that were fields of the ```DEBUG_EVENT```. The continue status is a bit less obvious. It can be used to suppress the normal exception handling behavior by passing ```DBG_EXCEPTION_HANDLED```. For now, we won't worry about that and just always pass in ```DBG_EXCEPTION_NOT_HANDLED```

```rust {linenos=true,lineNumbersInTable=false,linenostart=79}
        unsafe {
            ContinueDebugEvent(
                debug_event.dwProcessId,
                debug_event.dwThreadId,
                DBG_EXCEPTION_NOT_HANDLED,
            );
        }
```
<small>[main.rs line 79](https://github.com/TimMisiak/dbgrs/blob/part1/src/main.rs#L79)</small>

# Putting it together

Build the project using ```cargo build```, and then run it as ```dbgrs cmd``` and you'll see some output in the current window, as well as seeing a new command window open up. If you close that window, you'll see a few more lines of output, ending with an ExitProcess. For me, it looked like this:

```
D:\git\dbgrs\target\debug>dbgrs.exe cmd
Command line was: 'cmd'
CreateProcess
LoadDll
LoadDll
LoadDll
LoadDll
CreateThread
LoadDll
LoadDll
LoadDll
CreateThread
Exception
LoadDll
LoadDll
LoadDll
LoadDll
UnloadDll
UnloadDll
UnloadDll
ExitThread
ExitThread
ExitProcess
```

Each of those lines represents a single event returned from ```WaitForDebugEventEx```. Each of these events is also an opportunity for a debugger to examine the state of the process. We haven't yet started to look at the specific information from each event (like the name of the DLL that is loaded), but you can see how WinDbg would implement a command like ```sxe ld``` where the debugger can "break in" as soon as a module is loaded. All it means to "break in" when live debugging a process is simply wait before calling ```ContinueDebugEvent```. After one of these events is returned, we could allow an interactive user to run some commands to examine the state of a process. When they were done and executed a command like ```g``` to continue execution, the debugger will call ```ContinueDebugEvent``` and then ```WaitForDebugEventEx``` again to wait for the next event.

Most of these events look pretty reasonable. We'd expect that a bunch of DLLs get loaded, a few threads get created, and eventually we unload some DLLs and exit the threads and process. There is one event hiding in the middle there, and it's an Exception. That's the initial breakpoint from ```ntdll!LdrpDoDebuggerBreak```! As part of process startup, ntdll will trigger a breakpoint exception with an explicit ```int 3``` on x86<sup>5</sup>.

How we handle exception events will be one of the most important parts of debugging a live process. Everything from breakpoints, stepping, and application crashes are managed through the exception event. We'll need to spend some time making sure we handle that correctly, so I will leave that for the next post. Until then, let me know what you thought of this on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim)!

## Footnotes

<sup>1</sup> Many debuggers also integrate with some form of "time travel debugging" where a trace of execution is captured at various levels of granularity. While many of the concepts we talk about here also apply to time travel debugging, I'm going to skip talking about that here for the sake of brevity.

<sup>2</sup> Using this API will check if the calling process actually has enough privilege to attach to the requested process. You cannot typically attach to an elevated process or process running for another user from an unelevated process. There are also certain processes that even an elevated administrator process can't directly attach to. The documentation describes this as "it must be able to open the process for ```PROCESS_ALL_ACCESS```.

<sup>3</sup> There is a ```WaitForDebugEvent``` function (without the "Ex"), but this function will not correctly recieve Unicode strings that are generated from ```OutputDebugStringW```. We want Unicode strings, so we'll use the "Ex" version.

<sup>4</sup> This means that it's impossible for two separate modules to debug two separate targets from a single process. These are very old APIs that rely on process-wide state, and can NEVER be used by a module that expects to be loaded into an arbitrary process. Luckily this is rarely a problem in practice.

<sup>5</sup> This breakpoint is only triggered if the process is being debugged. This is detected by checking the "BeingDebugged" flag of the PEB. If you break into the process prior to the initial breakpoint and set that flag to "0", the initial breakpoint will be skipped.