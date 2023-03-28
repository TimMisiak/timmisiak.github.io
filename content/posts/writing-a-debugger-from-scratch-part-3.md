---
title: "Writing a Debugger From Scratch - DbgRs Part 3 - Reading Memory"
date: 2023-03-28T10:14:19-07:00
draft: false
---

(New to this series? Consider starting from [part 1](/posts/writing-a-debugger-from-scratch-part-1))

At the end of the [last post](/posts/writing-a-debugger-from-scratch-part-2), we had the ability to launch a program, step through instructions, and examine registers. We're still not quite at the point that we can call this a "debugger" but we're getting pretty close. In this part, we're going to start taking a implementing functionality to examine the memory of the target process. You can see the full code for this post in the [part3 branch on github](https://github.com/TimMisiak/dbgrs/tree/part3). If you see any mistakes or ways to improve the code, feel free to [create issues](https://github.com/TimMisiak/dbgrs/issues) in the GitHub repo or submit a PR. I've had a few folks contribute issues and PRs, which I really appreciate!

The simplest memory command we can implement will take an address as a parameter and display a hex dump of memory at that address. In WinDbg/NTSD, that command would be "db", which is one of the ["display memory" commands](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/d--da--db--dc--dd--dd--df--dp--dq--du--dw--dw--dyb--dyd--display-memor). We'll use the same abbreviation here. Before we can implement the functionality for "db", we need to extend our command parser to accept numerical expressions.

# Extending the command parser

Since the command parser is written using [Rust Sitter](https://github.com/hydro-project/rust-sitter), extending the parser to accept more complicated expressions is pretty straightforward. We'll rename the ```Expr``` enum to ```CommandExpr``` since we'll now have a new type of expression, which is one that can be evaluated as a numeric expression. We'll call that ```EvalExpr```.

```rust
    #[rust_sitter::language]
    pub enum CommandExpr {
        StepInto(#[rust_sitter::leaf(text = "t")] ()),
        Go(#[rust_sitter::leaf(text = "g")] ()),
        DisplayRegisters(#[rust_sitter::leaf(text = "r")] ()),
        DisplayBytes(#[rust_sitter::leaf(text = "db")] (), Box<EvalExpr>),
        Evaluate(#[rust_sitter::leaf(text = "?")] (), Box<EvalExpr>),
        Quit(#[rust_sitter::leaf(text = "q")] ()),
    }

    #[rust_sitter::language]
    pub enum EvalExpr {
        Number(#[rust_sitter::leaf(pattern = r"(\d+|0x[0-9a-fA-F]+)", transform = parse_int)] u64),
        #[rust_sitter::prec_left(1)]
        Add(
            Box<EvalExpr>,
            #[rust_sitter::leaf(text = "+")] (),
            Box<EvalExpr>,
        ),
    }
```
<small>[command.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/command.rs)</small>

In addition to adding the ```EvalExpr``` type, there are also two new commands which take an ```EvalExpr```. The first is the ```?``` command which evaluates an expression. It will be a useful starting point to test parsing and evaluation of ```EvalExpr```. The other command is ```db```, which displays bytes at the address corresponding to the evaluated expression. For our numerical expression, we'll accept hexidecimal numbers or decimal numbers (using a '0x' prefix to denote hexidecimal), and we'll allow simple addition of terms with "+". Note that the ```prec_left``` is to make the language unambiguous by saying that the operator is left-associative, so 1 + 2 + 3 means (1 + 2) + 3, which in our [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) means ```Add(Add(Number(1), Number(2)), Number(3))```.

The ```transform``` function needs to take a string and return an integer that gets used as the value for the Number enum, so we'll write a short function to do that conversion.

```rust
    fn parse_int(text: &str) -> u64 {
        let text = text.trim();
        if text.starts_with("0x") {
            let text = text.split_at(2).1;
            u64::from_str_radix(text, 16).unwrap()
        } else {
            text.parse().unwrap()
        }
    }
```
<small>[command.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/command.rs)</small>

# Evaluating expressions

For now, the evaluator doesn't support symbols or any operations except addition so it's pretty straightforward.

```rust
    pub fn evaluate_expression(expr: EvalExpr) -> u64 {
        match expr {
            EvalExpr::Number(x) => x,
            EvalExpr::Add(x, _, y) => evaluate_expression(*x) + evaluate_expression(*y),
        }
    }
```

And this is all we need to implement the functionality for ```CommandExpr::Evaluate```

```rust
    CommandExpr::Evaluate(_, expr) => {
        let val = eval::evaluate_expression(*expr);
        println!(" = 0x{:X}", val);
    }
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/main.rs)</small>

Let's test it out:

```
Command line was: 'cmd.exe /k "echo hello" '
CreateProcess
[CC04] 0x00007fff36e82680
> ? 10 + 10
 = 0x14
[CC04] 0x00007fff36e82680
> ? 0x10 + 10 
 = 0x1A
[CC04] 0x00007fff36e82680
> ? 0x10 + 0x10 
 = 0x20
[CC04] 0x00007fff36e82680
```

# Displaying memory

Now that we can parse numeric expressions, we can hook up our memory display command. We start by evaluating the expression given.
```rust
    CommandExpr::DisplayBytes(_, expr) => {
        let addr = eval::evaluate_expression(*expr);
        let mut buffer: [u8; 16] = [0; 16];
        let mut bytes_read: usize = 0;
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/main.rs)</small>

To read memory from another process, we use the [ReadProcessMemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-readprocessmemory) API, which takes a set of parameters describing the process, address, number of bytes to read, and a local buffer pointer/length.

```rust
        let result = unsafe {
            ReadProcessMemory(
                process,
                addr as *const c_void,
                buffer.as_mut_ptr() as *mut c_void,
                buffer.len(),
                &mut bytes_read as *mut usize,
            )
        };
        if result == 0 {
            println!("ReadProcessMemory failed");
        } else {
            for n in 0..bytes_read {
                print!("{:02X} ", buffer[n]);
            }
            println!();
        }
    }
```

Using the instruction pointer, we have an address that we can use to dump the code bytes that we're executing.

```
[CC04] 0x00007fff36e82680
> db 0x00007fff36e82680
48 83 EC 78 4C 8B C9 48 8B 05 62 99 11 00 48 85 
```

I can recognize these bytes as x64 assembly code, so it looks like this is working. (How did I recognize these bytes as code? I wrote about this in my post about [recognizing patterns in memory](recognizing-patterns.md)).

There are a few other places where reading memory can give us better information, but before we go any further we should wrap the ReadProcessMemory call so it's a bit easier and safer to use. We're going to need to read all sorts of data types of data, including things like UTF-16 strings<sup>1</sup>, and we don't want to see these verbose ReadProcessMemory calls everywhere. We'll also want to start creating a level of abstraction so that we can handle reading memory from other sources besides just a live process (for instance, crash dump files).

# MemorySource trait

We'll define a Rust [trait](https://doc.rust-lang.org/book/ch10-02-traits.html) that represents something that we can read memory from. Unlike dealing with memory within a process, reading memory from a different process often has to deal with "partial" reads where some part of the memory isn't available (such as an unmapped page, or memory missing in a crash dump). There are two ways to handle unavailable memory. You can either read as many contiguous bytes as are available, or you can return an array of items that represent the entire requested range with a flag on each one to indicate if it was available or not. Reading the largest contiguous range will be a function called ```read_raw_memory``` in this trait, and we will generally use this when we are reading data structures or strings out of the remote process. For displaying memory to a user with a ```db``` command, however, we want to indicate which bytes were available and which are not, and we will call this trait function ```read_memory```.

```rust
pub trait MemorySource {
    // Read up to "len" bytes, and stop at the first failure
    fn read_raw_memory(&self, address: u64, len: usize) -> Vec<u8>;
    // Read up to "len" bytes, and return Option<u8> to represent what bytes are available in the range
    fn read_memory(&self, address: u64, len: usize) -> Result<Vec<Option<u8>>, &'static str>;
}
```
<small>[memory.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/memory.rs)</small>

On top of this trait, we'll have some helper functions that let us read strings and other structures. I'll leave the definitions out for brevity (and since they're not particularly interesting), but you can check the github repo to see how these are implemented.<sup>2</sup>

```rust
pub fn read_memory_data<T: Sized + Default + Copy>(
    source: &dyn MemorySource,
    address: u64,
) -> Result<T, &'static str> { ... }
pub fn read_memory_array<T: Sized + Default>(
    source: &dyn MemorySource,
    address: u64,
    max_count: usize,
) -> Result<Vec<T>, &'static str> { ... }
pub fn read_memory_string(
    source: &dyn MemorySource,
    address: u64,
    max_count: usize,
    is_wide: bool,
) -> Result<String, &'static str> { ... }
```
<small>[memory.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/memory.rs)</small>

# Reading debug output strings

Now that we can easily read structures and strings from the remote process, we can improve our handling of a few debug events to give better information to users. The obvious one to start with is ```OUTPUT_DEBUG_STRING_EVENT``` which will tell us the text that a target process passes to [```OutputDebugStringW```](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-outputdebugstringw).

The ```OUTPUT_DEBUG_STRING_EVENT``` constant corresponds to the ```OUTPUT_DEBUG_STRING_INFO``` struct in the event union. Output strings could be windows "wide" or "ANSI" depending on if ```OutputDebugStringA``` or ```OutputDebugStringW``` is used. The ```fUnicode``` member tells us which to expect. The address is given in ```lpDebugStringData``` and this is an address in the target process address space. We also get the length of the string, which makes it convenient to know how much memory we should try to read. We can read the string using the ```read_memory_string``` helper that we just added.

```rust
    OUTPUT_DEBUG_STRING_EVENT => {
        let debug_string_info = unsafe { debug_event.u.DebugString };
        let is_wide = debug_string_info.fUnicode != 0;
        let address = debug_string_info.lpDebugStringData as u64;
        let len = debug_string_info.nDebugStringLength as usize;
        let debug_string =
            memory::read_memory_string(mem_source.as_ref(), address, len, is_wide).unwrap();
        println!("DebugOut: {}", debug_string);
    }
```

To test this, I wrote a short C++ program that calls the two debug output APIs.

```c++
#include <windows.h>

int main()
{
    OutputDebugStringA("Hello world!");
    OutputDebugStringW(L"Hello Unicode world!");
}
```

Running this under our debugger and we get the expected output. It's alive!

```
> g
DebugOut: Hello world!
[CDB8] 0x00007ffd3b6fcd29
> g
DebugOut: Hello Unicode world!
[CDB8] 0x00007ffd3b6fcd29
```

# Better module load notifications

Now that we have debug output strings displaying, we can move on to module load notifications. Right now, there isn't any useful information being displayed, but the Windows [WaitForDebugEventEx](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-waitfordebugeventex) API will helpfully provide a pointer to a module name as part of the ```LOAD_DLL_DEBUG_EVENT```. Reading the module name is slightly more complicated than the debug strings, because the provided field is actually the address of a pointer to the string, so we have an extra level of indirection to deal with, but otherwise is very similar. We can use the ```read_memory_data``` helper to grab the pointer, and then read the actual string using ```read_memory_string``` again. Note that the event does not tell us how long the string is, so we'll just assume that the modules won't be longer than ```MAX_PATH``` (260).

Note that the ```lpImageName``` field is not always provided, so we have to handle the case where this is null. There are other ways we can determine the name of a module from the provided information, but we'll leave that for the future.

```rust
    LOAD_DLL_DEBUG_EVENT => {
        let load_dll = unsafe { debug_event.u.LoadDll };
        let dll_base: u64 = load_dll.lpBaseOfDll as u64;
        if load_dll.lpImageName != std::ptr::null_mut() {
            let dll_name_address = memory::read_memory_data::<u64>(
                mem_source.as_ref(),
                load_dll.lpImageName as u64,
            )
            .unwrap();
            let is_wide = load_dll.fUnicode != 0;

            let dll_name = memory::read_memory_string(
                mem_source.as_ref(),
                dll_name_address,
                260,
                is_wide,
            )
            .unwrap();

            println!("LoadDll: {:X}   {}", dll_base, dll_name);
        } else {
            println!("LoadDll: {:X}", dll_base);
        };
    }
```
<small>[main.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/main.rs)</small>

Testing this again and we see the expected output!

```
Command line was: 'cmd.exe /k "echo hello" '
CreateProcess
[BD14] 0x00007ffd3d9e2680
> g
LoadDll: 7FFD3D990000    
[BD14] 0x00007ffd3d9e2680
> g
LoadDll: 7FFD3CD20000   C:\WINDOWS\System32\KERNEL32.DLL
[BD14] 0x00007ffd3da2d5c4
> g
LoadDll: 7FFD3B6D0000   C:\WINDOWS\System32\KERNELBASE.dll
[BD14] 0x00007ffd3da2d5c4
> g
LoadDll: 7FFD3D5B0000   C:\WINDOWS\System32\msvcrt.dll
[BD14] 0x00007ffd3da2d5c4
```

You might note that the very first module load is missing a name, and this is one of the cases where the provided ```lpImageName``` field is null. Using an instance of WinDbg with a [noninvasive attach](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/noninvasiv-debugging--user-mode-), we can use the [!dh](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/-dh) or [.imgscan](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/-imgscan--find-image-headers-) commands to examine the module at that address.

```
0:000> .imgscan
MZ at 00007ff7`cb470000, prot 00000002, type 01000000 - size 67000
  Name: cmd.exe
MZ at 00007ffd`3d990000, prot 00000002, type 01000000 - size 1f8000
  Name: ntdll.dll

  0:000> !dh 7FFD3D990000

File Type: DLL
FILE HEADER VALUES
    8664 machine (X64)
...
Debug Directories(4)
	Type       Size     Address  Pointer
	cv           22      140160   13d360	Format: RSDS, guid, 1, ntdll.pdb
```

From this output, we can see that the module is ntdll.dll, which is definitely a bit of a "special" module so it's not really a surprise that it's the only module with a missing name. We could add some additional logic to deduce the module name when it isn't returned as part of the ```LOAD_DLL_DEBUG_EVENT```, but we can revisit this once we do more interpretation of the memory in a module.

# Memory done, what's next?

At this point, we've created something that can read lots of different types of raw data from a target, but we still don't have a lot of interpretation happening. We need to interpret code bytes, modules, symbol names, and data types, to name a few. That's a lot to tackle, but next time we will start with a few of these. 

Are you enjoying this series? Have a question or suggestion? Let me know! You can find me on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim).

## Footnotes

<sup>1</sup> I always feel weird calling Windows wide strings "UTF-16", because that's not quite right, for a number of reasons (for instance, most Windows APIs are happy to accept invalid UTF-16, like invalid surrogate pairs). Usually, it's reasonable to just ignore the problem entirely and pretend that it's just UTF-16, but when writing diagnostic tools I think it's important to handle bad data in a way that doesn't hide information or simply fail. Someone could be debugging a problem related to text encoding, and it's important to give users enough information to diagnose the issue.

<sup>2</sup> I'm also not completely happy with how it's implemented. I suspect there is a safer, Rust-ier way to do it. You can see the code in [memory.rs](https://github.com/TimMisiak/dbgrs/blob/part3/src/memory.rs). Feel free to give suggestions on how to implement it better.