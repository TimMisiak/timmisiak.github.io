---
title: "Writing a Debugger From Scratch - DbgRs Part 2 - Register State and Stepping"
date: 2023-03-06T08:44:39-08:00
draft: false
---

When we left off [last time](/posts/writing-a-debugger-from-scratch-part-1), we had a basic "debugger" that could launch a Windows process and monitor events that occur in that process, but it's not yet something that you would *really* call a debugger. Two things that are missing are the ability to examine the state of the process and to control its execution. So that's what we're going to build next.

The code for this part is on GitHub as the [part2 branch](https://github.com/TimMisiak/dbgrs/tree/part2). If you see any mistakes or ways to improve the code, feel free to [create issues](https://github.com/TimMisiak/dbgrs/issues) in the GitHub repo or submit a PR. I've had a few folks contribute issues and PRs, which I really appreciate!

# Parsing commands

In order to interact with a debugger, we need some sort of user interface. To keep things simple, we'll create a console interface that will use some of the same command names as ntsd. To do that, we'll need some basic string parsing. The commands we're starting with are going to be very simple without any arguments or parameters, but in a future part we'll be adding more complicated commands. To get a head start on that, we'll use [Rust Sitter](https://github.com/hydro-project/rust-sitter), which seems like a nice simple way to write a parser. 

We'll start with four basic commands to implement: "step into", "go", "display registers", and "quit". These are some of the most basic commands that we can implement that do not require any interpretation of the state within the target process, and can be implemented purely in terms of reading and writing to the register context. 

```rust {linenos=true,lineNumbersInTable=false,linenostart=7}
#[rust_sitter::grammar("command")]
pub mod grammar {
    #[rust_sitter::language]
    pub enum Expr {
        StepInto(#[rust_sitter::leaf(text = "t")] ()),
        Go(#[rust_sitter::leaf(text = "g")] ()),
        DisplayRegisters(#[rust_sitter::leaf(text = "r")] ()),
        Quit(#[rust_sitter::leaf(text = "q")] ()),
    }
}
```
<small>[command.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/command.rs)</small>

We'll read input from stdin and attempt to parse it. If we fail to parse the input, we'll display the error and ask the user for input again. I've omitted some parts because the focus is not the error handling, but you can see the full code in command.rs in the GitHub repo.

```rust {linenos=true,lineNumbersInTable=false,linenostart=68}
pub fn read_command() -> grammar::Expr {
    let stdin = std::io::stdin();
    loop {
        print!("> ");
        std::io::stdout().flush().unwrap();
        let mut input = String::new();
        stdin.read_line(&mut input).unwrap();
        let input = input.trim().to_string();
        if !input.is_empty() {
            let cmd = grammar::parse(&input);
            match cmd {
                Ok(c) => return c,
                Err(errs) => {
                    // Omitted for brevity
                }
            }
        }
    }
}
```
<small>[command.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/command.rs)</small>


# Reading the registers

Now we have a way of reading commands from a user but we still need to retrieve the relevant information so we can display it. The commands we're implementing this time all revolve around the register context. Each thread has its own set of registers, so the API for reading the registers is called [GetThreadContext](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getthreadcontext). Note that what goes in a "context" can get very complex, especially when working with things like the AVX registers or other context extensions. We'll skip most of that for now, but if you want to read more about it you can consult the [documentation on XState](https://learn.microsoft.com/en-us/windows/win32/debug/working-with-xstate-context).

The ```GetThreadContext``` function takes two arguments. The first is a thread handle and it has to be opened with the ```THREAD_GET_CONTEXT``` flag. Later we'll also want to use ```SetThreadContext```, which will require the ```THREAD_SET_CONTEXT``` flag, so we'll use both when we open the thread handle. We can use the thread ID we got from the ```WaitForDebugEventEx```. Thread IDs are unique across the entire system, so no process ID is needed here. 
I've also made a little utility container called ```AutoClosedHandle``` to make sure that ```CloseHandle``` gets called on the handle when it gets dropped.

```rust {linenos=true,lineNumbersInTable=false,linenostart=137}
        let thread = AutoClosedHandle(unsafe {
            OpenThread(
                THREAD_GET_CONTEXT | THREAD_SET_CONTEXT,
                FALSE,
                debug_event.dwThreadId,
            )
        });
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/main.rs)</small>

The second argument to ```GetThreadContext``` is the win32 ```CONTEXT``` structure. This is a bit of an odd structure on Windows, because it's specific to a CPU architecture, and the version you get is determined at compile time by the architecture of the debugger process, not the target process. As you might imagine, this can make cross-architecture debugging somewhat complicated. We'll leave that problem for another day and just assume that this is an x64 debugger process debugging an x64 target process. The definition of ```CONTEXT``` also has some alignment requirements, which are unfortunately not automatically annotated by the windows-sys crate right now. The folks on the windows-rs project [are aware of the issue](https://github.com/microsoft/windows-rs/issues/2347), but for now we can work around this by wrapping the structure inside another structure that has proper alignment.

```rust {linenos=true,lineNumbersInTable=false,linenostart=31}
#[repr(align(16))]
struct AlignedContext {
    context: CONTEXT,
}
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/main.rs)</small>

Also, because the context parameter to ```GetThreadContext``` is an in/out parameter, we need to make sure it's properly initialized, including the ```ContextFlags``` field which determines which flags should be queried (and which are valid when we call ```SetThreadContext```). With that, we can finally call ```GetThreadContext```.

```rust {linenos=true,lineNumbersInTable=false,linenostart=144}
        let mut ctx: AlignedContext = unsafe { std::mem::zeroed() };
        ctx.context.ContextFlags = CONTEXT_ALL;
        let ret = unsafe { GetThreadContext(thread.handle(), &mut ctx.context) };
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/main.rs)</small>

# Handling commands

Using the register context, we can display a summary of the current state and a prompt for the user to enter commands. Since we don't have any kind of symbol resolution yet, we'll have to settle for displaying the current thread ID and the current instruction pointer.

```rust {linenos=true,lineNumbersInTable=false,linenostart=152}
        let mut continue_execution = false;

        while !continue_execution {
            println!("[{:X}] {:#018x}", debug_event.dwThreadId, ctx.context.Rip);
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/main.rs)</small>

Next we'll read a command from the prompt and have handlers for each of the commands. We'll start with ```Expr::DisplayRegisters```, which has a straightforward implementation:

```rust {linenos=true,lineNumbersInTable=false,linenostart=159}
            let cmd = command::read_command();
            match cmd {
                Expr::DisplayRegisters(_) => {
                    registers::display_all(ctx.context);
                }
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/main.rs)</small>

```rust {linenos=true,lineNumbersInTable=false,linenostart=5}
pub fn display_all(context: CONTEXT) {
    println!("rax={:#018x} rbx={:#018x} rcx={:#018x}", context.Rax, context.Rbx, context.Rcx);
    println!("rdx={:#018x} rsi={:#018x} rdi={:#018x}", context.Rdx, context.Rsi, context.Rdi);
    println!("rip={:#018x} rsp={:#018x} rbp={:#018x}", context.Rip, context.Rsp, context.Rbp);
    println!(" r8={:#018x}  r9={:#018x} r10={:#018x}", context.R8, context.R9, context.R10);
    println!("r11={:#018x} r12={:#018x} r13={:#018x}", context.R11, context.R12, context.R13);
    println!("r14={:#018x} r15={:#018x} eflags={:#010x}", context.R14, context.R15, context.EFlags);
}
```
<small>[registers.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/registers.rs)</small>


The ```Expr::Go``` and ```Expr::Quit``` commands are also straightforward:

```rust {linenos=true,lineNumbersInTable=false,linenostart=169}
                Expr::Go(_) => {
                    // We'll break out of the loop and call ContinueDebugEvent
                    continue_execution = true;
                }
                Expr::Quit(_) => {
                    // The process will be terminated since we didn't detach.
                    return;
                }
```

## Step command

The simplest type of step command is the "step into" command. We don't have any source line information, so this will specifically be an instruction-level step into, which simply means that we will ask the thread to execute a single instruction. Luckily, most CPUs have a mechanism for this, and on x86 this is called the "trap flag" or TF for short. We'll take the context that we just read from the thread, add the trap flag to the EFLAGS register, and then set the context on the current thread.

```rust {linenos=true,lineNumbersInTable=false,linenostart=160}
                Expr::StepInto(_) => {
                    ctx.context.EFlags |= TRAP_FLAG;
                    let ret = unsafe { SetThreadContext(thread.handle(), &ctx.context) };
                    if ret == 0 {
                        panic!("SetThreadContext failed");
                    }
                    expect_step_exception = true;
                    continue_execution = true;
                }
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/main.rs)</small>

When execution is resumed on the thread, it will allow a single instruction to execute before it causes a "trap", which is represented as an exception debug event. But there's a small twist here, because if you try to use the step command with what we've written so far, it will look like the step command worked but the process will exit unexpectedly instead of letting you continue to step. This is because the trap flag has created an unhandled exception that the program was not designed to handle, which results in the process terminating. Luckily the ContinueDebugEvent takes a parameter of ```dwContinueStatus``` where passing ```DBG_CONTINUE``` tells the kernel to mark the exception as "handled" and causes the execution to continue normally at the location of the current thread context. We need to pass that value only when we know the exception was caused by a step command, or else we will end up suppressing real exceptions.

We could simply assume that any exception with code 0x80000004 ([EXCEPTION_SINGLE_STEP](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-exception_record)) was one that was generated by the debugger, and I'm sure some debuggers do that. However, single step exceptions can happen normally in a program without a debugger attached (either intentionally or due to program error), and are occasionally used as a form of anti-debugging. If we're not careful, we could change the program behavior unintentionally while debugging it. To keep things simple, we'll just assume that the next exception with code EXCEPTION_SINGLE_STEP *after* a step operation is one that was caused by the debugger.

```rust {linenos=true,lineNumbersInTable=false,linenostart=109}
            EXCEPTION_DEBUG_EVENT => {
                let code = unsafe { debug_event.u.Exception.ExceptionRecord.ExceptionCode };
                let first_chance = unsafe { debug_event.u.Exception.dwFirstChance };
                let chance_string = if first_chance == 0 {
                    "second chance"
                } else {
                    "first chance"
                };

                if expect_step_exception && code == EXCEPTION_SINGLE_STEP {
                    expect_step_exception = false;
                    continue_status = DBG_CONTINUE;
                } else {
                    println!("Exception code {:x} ({})", code, chance_string);
                    continue_status = DBG_EXCEPTION_NOT_HANDLED;
                }
            }
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part2/src/main.rs)</small>

# Trying it out

Let's run the debugger and see how it looks:

```
Command line was: 'cmd.exe /k "echo hello" '
CreateProcess
[24EC] 0x00007ff812d42680
> g
LoadDll
[24EC] 0x00007ff812d42680
> g
LoadDll
[24EC] 0x00007ff812d8d5c4
> g
LoadDll
[24EC] 0x00007ff812d8d5c4
> t
[24EC] 0x00007ff812d8d5c4
> t
[24EC] 0x00007ff812d04d42
> t
[24EC] 0x00007ff812d04d44
> r
rax=0x0000000000000000 rbx=0x0000000000000000 rcx=0x00007ff812d8d5c4
rdx=0x0000000000000000 rsi=0x000001ee3e0647c0 rdi=0x000001ee3e064680
rip=0x00007ff812d04d44 rsp=0x00000063ed4fe750 rbp=0x00000063ed4fe7d0
 r8=0x00000063ed4fe748  r9=0x00000063ed4fe7d0 r10=0x0000000000000000
r11=0x0000000000000344 r12=0xffffffffffffffff r13=0x00000063ed2b0000
r14=0x000001ee3e064700 r15=0x0000000000800000 eflags=0x00000246
[24EC] 0x00007ff812d04d44
> t
[24EC] 0x00007ff812d04d48
> r
rax=0x0000000000000000 rbx=0x0000000000000000 rcx=0x00007ff812d8d5c4
rdx=0x0000000000000000 rsi=0x000001ee3e0647c0 rdi=0x000001ee3e064680
rip=0x00007ff812d04d48 rsp=0x00000063ed4fe750 rbp=0x00000063ed4fe7d0
 r8=0x00000063ed4fe748  r9=0x00000063ed4fe7d0 r10=0x0000000000000000
r11=0x0000000000000344 r12=0xffffffffffffffff r13=0x00000063ed2b0000
r14=0x000001ee3e064700 r15=0x0000000000800000 eflags=0x00000246
[24EC] 0x00007ff812d04d48
> q
```

What we have now is something incredibly close to what we could really call "a debugger". You can continue execution, you can step through code, and you can examine the registers. There are a few really big things missing that prevent this from actually being useful, and most of them have to do with the memory of the target process. This debugger can't read or write from the targets memory, and that means it can't do a number of other important things like resolving symbolic names or displaying disassembly. We'll start to tackle that in the next part. Until then, let me know what you thought of this on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim)!

# We're hiring!

If you read this far, you should know that my company is hiring! I started [Augmend](https://augmend.com) with [Diamond Bishop](https://twitter.com/diamondbishop) to make AI-augmented systems for software engineers to collaborate. We think that making it easier to collaborate with both people and AI assistants is going to speed up the entire software lifecycle, from writing code, testing code, debugging code, and more. Excited about dev tools and AI? Check out our [careers page](https://augmend.com/careers) and see if there's a role that interests you!