---
title: "Writing a Debugger From Scratch - DbgRs Part 8 - Source and Symbols"
date: 2024-05-29T08:53:13-07:00
draft: true
---

(New to this series? Consider starting from [part 1](/posts/writing-a-debugger-from-scratch-part-1))

At the end of the [last post](/posts/writing-a-debugger-from-scratch-part-7), DbgRs got the ability to disassemble the code that it was debugging. While disassembly is critical for many debugging tasks, using source code to step through the code line by line is usually more convenient when it's available. In this post we'll start looking at how to use symbols (PDB files on Windows) so that we can display the source for the code that's being debugged.

The DbgRs code for this post is in the [part8 branch on github](https://github.com/TimMisiak/dbgrs/tree/part8). You can also view the [changes from part7](https://github.com/TimMisiak/dbgrs/compare/part7...part8). If you spot any mistakes or have suggestions for improving the code, feel free to [create issues](https://github.com/TimMisiak/dbgrs/issues) on the GitHub repo or submit a PR.


# What's in a PDB?

A symbol file is the debugger's link between the compiled code and the original source code. Not just the correlation between the lines of code and the assembly instructions, but also function signatures, type definitions, local variables, and more. In the Windows world, this file is called a "PDB File", which [stands for Program DataBase](https://web.archive.org/web/20160324115412/https://support.microsoft.com/en-us/kb/121366). (Previously, debug information was stored directly in the executable, or <a aria-describedby="footnote-label" href="#dbg-files">optionally in a .DBG file</a> but this is a long-dead format). These days, there are a number of different symbol formats known as "PDB", including the [.NET "Portable PDB"](https://github.com/dotnet/runtime/blob/main/docs/design/specs/PortablePdb-Metadata.md) format, as well as the ["Fastlink" PDB](https://devblogs.microsoft.com/cppblog/faster-c-build-cycle-in-vs-15-with-debugfastlink/) format that relies on the debug information on the original .OBJ files. For the purpose of this blog post, however, we will ignore those and focus on the "original" flavor of PDB.

While the PDB format is not documented, Microsoft released the [Microsoft-PDB repo](https://github.com/microsoft/microsoft-pdb/) which contains the source code to a number of key tools that were used for generating and parsing the PDB format. That's been sufficient for a number of other projects including LLVM to create PDB files that are compatible with WinDbg and Visual Studio. It's also been the basis for some open source projects that parse Windows PDB files including the [pdb crate](https://crates.io/crates/pdb) used by DbgRs.

I'm not going to cover the file format in depth, but I think it's useful to understand the rough outline of <a aria-describedby="footnote-label" href="#llvm-pdb-docs">what a PDB contains</a>. At the top level you have the [MSF](https://llvm.org/docs/PDB/MsfFile.html) (Multi-Stream File) container that allows a file to be broken up into a collection of numbered streams. It's a sort of mini-filesystem where individual blocks of the file can be allocated to different streams. Several fixed stream indices are used (streams 0-4), and one of these streams (stream 1, the "PDB stream") includes a name/index mapping to allow streams to have a <a aria-describedby="footnote-label" href="#named-streams">name in addition to an index</a>.

Some of the information, like the type description stream, is global for the entire binary and is stored in a single named or indexed stream. The rest of the information is stored on a per-module basis, where the "modules" are the individual compilation units (like OBJ and LIB files) that were statically linked together to create the binary. The list of modules is stored in the "DBI stream" (index 3) and each module has an MSF stream where the symbol information is stored. That stream of data is split into two important parts: The "Symbol Table" and the "Line Data".

## The Symbol Table

The Symbol Table is a sequence of variable-length <a aria-describedby="footnote-label" href="#codeview-records">records</a>, each with a type field that indicates what sort of information is contained. For instance, the S_GPROC32 indicates the start of a global procedure. There are more than a hundred <a href="https://github.com/microsoft/microsoft-pdb/blob/805655a28bd8198004be2ac27e6e0290121a5e89/include/cvinfo.h#L2736">different types of records</a>. The ordering of the records is important, as there are "start" and "end" records of various types. For instance, there can be various records annotating information about a procedure in between an S_GPROC32 record and the terminating S_END record. These child records can describe exception handlers, local variables, parameters, and more.

We'll look more at the other record types later, but we can start by just looking at the S_GPROC32 and similar entries to get better name resolution in our expression evaluator. Our current name to address resolution only works for export functions, which is pretty inconvenient. We'll start by factoring out the code for searching the exports, and adding a new function that looks at the symbol information:

```rust
pub fn resolve_function_in_module(module: &mut Module, func: &str) -> Option<u64> {
    // We'll search exports first and private symbols next
    let export_resolution = resolve_export_in_module(module, func);
    if export_resolution.is_some() {
        return export_resolution;
    }

    resolve_symbol_name_in_module(module, func).unwrap_or(None)
}
```

To resolve a name to an address, we'll need to get the list of modules (compilation units) described by the DBI stream. We'll also need the "address map" that allows mapping between a section index + offset in the original unoptimized binary to the final RVA as it exists in the binary we are debugging.

```rust
fn resolve_symbol_name_in_module(module: &mut Module, func: &str) -> Result<Option<u64>, anyhow::Error> {
    let pdb = module.pdb.as_mut().ok_or(anyhow!("No PDB loaded"))?;
    let dbi = pdb.debug_information()?;
    let mut modules = dbi.modules()?;
    let address_map = module.address_map.as_mut().ok_or(anyhow!("No address map available"))?;
```

Given a function name, we don't know which compilation unit it came from, so we'll search through each module in order, and for each module we'll iterate over the contained symbol records.

```rust
    while let Some(pdb_module) = modules.next()? {
        let mi = pdb.module_info(&pdb_module)?.ok_or(anyhow!("Couldn't get module info"))?;
        let mut symbols = mi.symbols()?;
        while let Some(sym) = symbols.next()? {
```

The pdb crate we're using has a nice way of representing symbol data so that various related records like S_GPROC32 and S_LPROC32 can all be represented as a single logical type. So we can look for anything that's a "Procedure" and check if it has a name that matches the name we are looking for. If we find it, we'll convert the section index + offset into an RVA, and then convert that to a virtual address by adding the module base.

```rust
            if let Ok(parsed) = sym.parse() {
                if let SymbolData::Procedure(proc_data) = parsed {
                    if proc_data.name.to_string() == func {
                        let rva = proc_data.offset.to_rva(address_map).ok_or(anyhow!("Couldn't convert procedure offset to RVA"))?;
                        let address = module.address + rva.0 as u64;
                        return Ok(Some(address));
                    }
                }
            }
        }
    }
    Ok(None)
}
```


We already had the resolve_function_in_module function connected to the <a aria-describedby="footnote-label" href="#other-eval-changes">expression evaluator</a>, so we should be able to use this with the "bp" command right away. Let's try it.

```
Command line was: '"C:\git\HelloWorld\hello.exe" '
LoadDll: 7FF7AB070000   hello.exe
[87F0] 0x00007ffecfe2aa20
> bp hello!main
[87F0] 0x00007ffecfe2aa20
> g
LoadDll: 7FFECFDD0000   ntdll.dll
[87F0] ntdll.dll!RtlUserThreadStart
> g
LoadDll: 7FFECE9F0000   C:\Windows\System32\KERNEL32.DLL
[87F0] ntdll.dll!NtMapViewOfSection+0x14
> g
LoadDll: 7FFECD280000   
[87F0] ntdll.dll!NtMapViewOfSection+0x14
> g
Exception code 80000003 (first chance)
[87F0] ntdll.dll!LdrInitShimEngineDynamic+0x345
> g
Breakpoint 0 hit
[87F0] hello.exe!main
> 
```

With that out of the way, we can start looking at how to use the line data to map between code addresses and source lines.

## Mapping Addresses to Line Numbers

The information about line number mapping is stored in the "Line Data" (aka [line number debug info](https://github.com/microsoft/microsoft-pdb/blob/master/PDB/dbi/dbi.h#L1146)), which is in the MSF stream for the module, immediately following the symbol table. We can use this information to map a code address like the current instruction pointer to a file name and line number. Using that we can load the corresponding source file and display each line to the user as they step through the code. We'll start by grabbing the `pdb` object that gives us access to the contents of the symbol file.

```rust
pub fn resolve_address_to_source_line(address: u64, process: &mut Process) -> Result<(String, u32)> {
    let module = process.get_containing_module_mut(address).ok_or(anyhow!("Module not found"))?;
    let pdb = module.pdb.as_mut().ok_or(anyhow!("Symbols not available"))?;
```

The source line information, like most data in the PDB file, is stored in terms of the original section number and section offset, so we first need to convert from an address in the process to an RVA, and then from an RVA to the original section + offset address, which the PDB module calls an "internal offset".

```rust
    let address_map = module.address_map.as_mut().ok_or(anyhow!("Address map not found for module"))?;
    let rva: u32 = (address - module.address).try_into()?;
    let rva = Rva(rva);
    let offset = rva.to_internal_offset(address_map).ok_or(anyhow!("Couldn't map address"))?;
```

Next, we'll <a aria-describedby="footnote-label" href="#faster-source-lookups">enumerate over all the compilation units/modules</a> looking for one that contains a line mapping for the offset that we just found.

```rust
    let dbi = pdb.debug_information()?;
    let mut modules = dbi.modules()?;
    while let Some(module) = modules.next()? {
        if let Ok(Some(mi)) = pdb.module_info(&module) {
            if let Ok(lp) = mi.line_program() {
```

The PDB stores line mappings in chunks that correspond to functions or parts of functions. We can use the pdb to get the group of line mappings that corresponds to a particular address, but we'll still have to enumerate over the line mappings to find the right one. Source lines are not one-to-one mapped with code addresses. A single line of source typically compiles to multiple assembly instructions. Beyond that, optimizations can cause more ambiguity when resolving a source line to an address (more on that later). But mapping from a code address to a source line is <a aria-describedby="footnote-label" href="#code-to-line-duplicates">mostly unambiguous</a>, and we can simply find the first line entry that contains the code address we're looking for.

```rust
                let mut lines = lp.lines_for_symbol(offset);
                while let Some(line) = lines.next()? {
                    if line.offset.offset <= offset.offset {
                        let diff = offset.offset - line.offset.offset;
                        if diff < line.length.unwrap_or(0) {
```

Once we find the right one, we can look up the information about the source file in a separate table, and then use information in that structure to get the actual name of the file from the "string table" (used by the PDB format to deduplicate common strings). And at this point, we have a source file and line number to return.

```rust
                            let file_info = lp.get_file_info(line.file_index)?;
                            let string_table = pdb.string_table()?;
                            let file_name = string_table.get(file_info.name)?;
                            return Ok((file_name.to_string().into(), line.line_start))
                        }
                        
                    }
                }
            }
        }
    }
    Err(anyhow!("Address not found"))
}

```


Using this we'll hook up a simple "lsa" command.

```rust
    CommandExpr::ListSource(_, expr) => {
        if let Some(val) = eval_expr(expr) {
            match resolve_address_to_source_line(val, &mut process) {
                Ok((file_name, line_number)) => {
                    println!("LSA: {}:{}", file_name, line_number);
                    if let Ok(file) = File::open(&file_name) {
                        let reader = io::BufReader::new(file);
                        let lines: Vec<_> = reader.lines().map(|l| l.unwrap_or("".to_string())) .collect();
```

And to make sure there's enough context, we'll print a small window of source lines around the target line. One important thing to remember is that line numbers are not 0-indexed, since the first line of a file is line 1!

```rust
                        for print_line_num in (max(1, line_number - 2))..=(min(lines.len() as u32, line_number + 2)) {
                            if print_line_num == line_number {
                                println!(">{:4}: {}", print_line_num, lines[print_line_num as usize - 1]);
                            } else {
                                println!("{:5}: {}", print_line_num, lines[print_line_num as usize - 1]);
                            }
                        }
                    } else {
                        println!("Couldn't open file: {}", file_name);
                    }
                },
                Err(e) => {
                    println!("Couldn't look up source: {}", e);
                }
            }                        
        }
    }
```

Now we get back source files and line numbers!

```
> lsa hello!main
LSA: C:\git\HelloWorld\hello.cpp:35
   33: 
   34: int main()
>  35: {
   36:     printf("Hello world, %d", func1());
   37:     printf("Hello world 2!");
```

The source file came back as an absolute path, so we were able to find the file that was just compiled locally. But some compilation settings will result in a relative path being stored, and binaries that were compiled on a different machine likely won't have the right source path for the local machine even if it is an absolute path. If we try to list the source for one of the CRT functions, we get this.

```
> lsa hello!common_lseek_do_seek_nolock
LSA: minkernel\crts\ucrt\src\appcrt\lowio\lseek.cpp:22
Couldn't open file: minkernel\crts\ucrt\src\appcrt\lowio\lseek.cpp
```

To get this working, we'll need to implement source search functionality.

### Source search

The relative path found in the symbols for the UCRT source file can't be opened directly, even though I do have the source file available in my SDK install folder. We can get these paths to match up by creating a source search algorithm that takes a set of search paths as potential "roots" to combine with the source path we are looking for. First, we'll add a ".srcpath" command that can set a Vec of search paths.

```rust
    CommandExpr::SrcPath(_, path) => {
        source_search_paths.clear();
        source_search_paths.extend(path.split(';').map(|s| s.to_string()));
    }
```

Now given that list of search paths, we need to find the correct matching path. If the path is an absolute path and exists, we'll just return that source file. Otherwise, we'll start iterating through the list of search paths. For each one, we'll start with the full starting path minus any drive specifier, e.g. "minkernel\crts\ucrt\src\appcrt\lowio\lseek.cpp" and append it to the search path. If the file isn't found, we'll start removing path elements from the front, one by one, looking for existing files on disk, until we're left with just the filename appended to the search path. That way, a search path can point to different levels of the filesystem and still find the right source file as long as some trailing part of the original path matches a local file. So even if the original path was on "Z:\foo\bar\baz\file.cpp", we can store the files at "C:\src\baz\file.cpp" and have a search path of "C:\src" and the file will be found.

```rust
pub fn find_source_file_match(file: &str, search_paths: &Vec<String>) -> Result<PathBuf> {
    let file_path = Path::new(file);

    // If the file path is absolute and exists, return it immediately.
    if file_path.is_absolute() && file_path.exists() {
        return Ok(file_path.to_path_buf());
    }

    // Get all subsets of the input path.
    let components: Vec<&str> = file_path.components().map(|c| c.as_os_str().to_str().unwrap()).collect();
    
    for search_path in search_paths {
        let search_path = Path::new(search_path);
        
        for i in 0..components.len() {
            // Join the search path with the subset of the input path.
            let test_path: PathBuf = search_path.join(components[i..].iter().collect::<PathBuf>());
            if test_path.exists() {
                return Ok(test_path.to_path_buf());
            }
        }
    }

    Err(anyhow!("File not found"))
}
```

If we go back to the case that failed earlier and add a search path pointing to the ucrt source in my local installation of the SDK, we'll see that DbgRs can now find this file.

```
> .srcpath C:\Program Files (x86)\Windows Kits\10\Source\10.0.22000.0\ucrt 
[8AC8] 0x00007ffecfe2aa20
> lsa hello!common_lseek_do_seek_nolock
LSA: minkernel\crts\ucrt\src\appcrt\lowio\lseek.cpp:22
Found matching file: C:\Program Files (x86)\Windows Kits\10\Source\10.0.22000.0\ucrt\lowio\lseek.cpp
   20: // end up moving out of range of 32-bit values.
   21: static long __cdecl common_lseek_do_seek_nolock(HANDLE const os_handle, long const offset, int const origin, __crt_cached_ptd_host& ptd) throw()
>  22: {
   23:     LARGE_INTEGER const origin_pos = { 0 };
   24: 
```

It works! Now there is one more problem here, and that is that the content of the file could be totally wrong. Maybe it's the wrong commit of a repository, or the filename just happens to match and is an unrelated file. Fortunately, the PDB file includes a hash of the source file as it was compiled. There are a number of different hash algorithms that can be used, but new compilers will default to a SHA256 hash. So we can pass that hash along to the find_source_file_match function and it can use that to reject any source files without a matching hash. But for the purposes of DbgRs, I'll leave that functionality out. (Feel free to make a PR if you'd like to contribute!)

## Mapping line numbers to addresses

While we're stepping through the code, we need to map each address to the corresponding source line, but when setting a source-level breakpoint, we need to do the opposite and map a source line to an address. To do that we'll extend the expression evaluator to take a syntax that allows us to specify the module, source file name, and line number, and allow that to resolve back to an address that we can use for a breakpoint. This is going to be a bit of a verbose syntax to type on the command line, but if DbgRs gets integrated into a GUI like VsCode, all that information <a aria-describedby="footnote-label" href="#source-file-info">will be available in the UI layer</a> to pass along to the debugging engline.

The syntax for specifying a source line in WinDbg involves using the `@@masm` syntax, which is a bit arcane:

```
0:000> ? @@masm(`notepad__!C:\git\npp\PowerEditor\src\winmain.cpp:721`)
Evaluate expression: 140697038077034 = 00007ff6`94f9bc6a
```

We'll simplify that a bit to get rid of the "masm" part but otherwise leave it mostly the same. We can add a new case for our EvalExpr enumeration, and add a regular expression that tries to parse something that looks roughly like "module!file:line".

```rust
    #[rust_sitter::language]
    pub enum EvalExpr {
        Number(#[rust_sitter::leaf(pattern = r"(\d+|0x[0-9a-fA-F]+)", transform = parse_int)] u64),
        Symbol(#[rust_sitter::leaf(pattern = r"(([a-zA-Z0-9_@#.]+!)?[a-zA-Z0-9_@#.]+)", transform = parse_sym)] String),
        SourceLine(#[rust_sitter::leaf(pattern = r"(`[^`!]+!(?:[a-zA-Z]+:)[^`:!]+:\d+`)", transform = parse_source_line)] (String, String, u32)),
```

We'll also need to extend the expression evaluator to handle this case:

```rust
pub fn evaluate_expression(expr: EvalExpr, context: &mut EvalContext) -> Result<u64, anyhow::Error> {
    match expr {
...
        EvalExpr::SourceLine((src_module, src_file, src_line)) => {
            resolve_source_line_to_address(&src_module, &src_file, src_line, context.process)
        }
```

To implement `resolve_source_line_to_address`, we'll search through all compilation units to find any that reference the source file that we're looking at. 

```rust
pub fn resolve_source_line_to_address(module_name: &str, src_file: &str, src_line: u32, process: &mut Process) -> Result<u64> {
...
    let mut modules = dbi.modules()?;
    while let Some(module) = modules.next()? {
        if let Ok(Some(mi)) = pdb.module_info(&module) {
            if let Ok(line_program) = mi.line_program() {
                if line_program_references_file(&line_program, src_file, &string_table)? {
```

The implementation of line_program_references_file is just a straight linear search through the list of files returned from the LineProgram (class provided by the pdb create).

```rust
fn line_program_references_file(line_program: &LineProgram, src_file: &str, string_table: &StringTable) -> Result<bool> {
    let mut files = line_program.files();
    while let Some(file) = files.next()? {
        let cur_file_name = string_table.get(file.name)?.to_string();
        if cur_file_name == src_file {
            return Ok(true);
        }
    }
    Ok(false)
}
```

For any compilation unit that references the source file in question, we'll search through the line mappings to find one that matches. There are faster ways of doing this search, but I don't see an easy way to do better than a linear search with the pdb crate. But it's fast enough for our purposes and is fairly easy to read.

```rust
                    let mut lines = line_program.lines();
                    while let Some(line) = lines.next()? {
                        let cur_file_name_ref = line_program.get_file_info(line.file_index)?.name;
                        let cur_file_name = string_table.get(cur_file_name_ref)?.to_string();
                        if cur_file_name == src_file && line.line_start <= src_line && src_line <= line.line_end {
                            let rva = line.offset.to_rva(&address_map).ok_or(anyhow!("Could not map source entry to RVA"))?;
                            let address = process_module.address + rva.0 as u64;
                            return Ok(address);
                        }
                    }
```

It's important to note again that the mapping from source line to address is not a one-to-one mapping. A single source line nearly always compiles to multiple assembly instructions. So in this case we're returning the first instruction in that block of instructions (which is typically what you want for a breakpoint). The instructions that result from a single line of code being compiled are also not necessarily contiguous, so you could have a few assembly instructions for line 1, a few instructions for line 2, and then some more for line 1 again. And beyond that, there's also cases where a function gets inlined, so a single source line can map to hundreds or thousands of addresses! For the sake of simplicity, we'll ignore all of that and simply return the first match we find. In a *real* debugger, you could not ignore that problem or you would have very unhappy users!

Now that we can resolve source lines to addresses and vice versa, we can easily test that we can "round-trip" a source line to an address and back to a source line.

```
> lsa `hello!C:\git\HelloWorld\hello.cpp:35`  
LSA: C:\git\HelloWorld\hello.cpp:35
Found matching file: C:\git\HelloWorld\hello.cpp
   33: 
   34: int main()
>  35: {
   36:     printf("Hello world, %d", func1());
   37:     printf("Hello world 2!");
```

It works! We can even set breakpoints using this:

```
> bp `hello!C:\git\HelloWorld\hello.cpp:37`  
[6DE8] 0x00007ffb0ebaaa20
> bl
  0 0x00007ff7f8f27207 (hello.exe!main+0x17)
[6DE8] 0x00007ffb0ebaaa20
```

That's a big step towards being useful for people that want to do more than read assembly code! But unfortunately, all of our data is still in registers and memory, so we're still not a "source level" debugger yet. That will have to be the next project, which will use the symbolic information to show information about local variables, with names and type information.

Hope you enjoyed this post about DbgRs! Have a question or suggestion? Let me know! You can find me on [Twitter](https://twitter.com/timmisiak), [Mastodon](https://dbg.social/@tim), and [Bluesky](https://bsky.app/profile/timdbg.com).

<footer>
  <h2 id="footnote-label">Footnotes</h2>
  <ol>
  <li id="dbg-files">
  The original .DBG format dates <a href="https://web.archive.org/web/20160324115412/https://support.microsoft.com/en-us/kb/121366">back to the DOS days</a>, when debugging information could not be appended directly to a .COM file, as this sort of executable file was loaded directly into memory and had to be smaller than 64K. So to provide debug information for .COM files, a separate .DBG file could be generated alongside the .COM file. Even back then, the debugging information was known as CodeView. I'm not sure if it has anything in common with the newer format sharing the same name.
  </li>
  <li id="llvm-pdb-docs">
  A good high level overview that goes into more depth is availbe in the <a href="https://llvm.org/docs/PDB/index.html">LLVM documentation for PDB files</a>. It's not exhaustive documentation, but it's a good starting point and was a reference I used when writing this blog post as well as a project I had to re-implement the pdbstr utility. There are <a href="https://github.com/llvm/llvm-project/issues/76602">some errors</a> and probably isn't actively being updated, but it's still the most accessible place to start.
  </li>
  <li id="named-streams">
  Any arbitrary named stream can be added to a PDB file using the <a href="https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/the-pdbstr-tool">pdbstr.exe tool</a>. This functionality is what allowed the original <a href="https://learn.microsoft.com/en-us/windows/win32/debug/source-server-and-source-indexing">"Source Indexing"</a> functionality to work, and is also used by the newer <a href="https://github.com/dotnet/sourcelink/blob/main/README.md">"Source Link"</a> format, both of which allow embedding information in a PDB file for locating the original version of a source file from a source repository or other server.
  </li>
  <li id="codeview-records">
  The records in the symbol table are sometimes just called "symbols", but I typically try to call them "symbol records" or "codeview records" because the term "symbol" is too overloaded of a term.
  </li>
  <li id="other-eval-changes">
  You might notice that I also changed the evaluator to work with module names with the extension omitted. This matches the windbg behavior (and thus my muscle memory).
  </li>
  <li id="faster-source-lookups">
  If I remember correctly, there is a faster way to lookup symbols than enumerating every module, as the PDB file has some embedded hash tables for direct lookup. Using a higher level mechanism like <a href="https://learn.microsoft.com/en-us/visualstudio/debugger/debug-interface-access/debug-interface-access-sdk?view=vs-2022">DIA</a> would probably use the embedded hashtables where possible (e.g. when not doing a pattern-based search). Similarly for source files, I believe there is a faster way to do the lookup for the correct module but the brute force way seems fast enough for our purposes and is pretty easy to understand. At least the operation for each module is fast, since lines_for_symbol is a binary search.
  </li>
  <li id ="code-to-line-duplicates">
  You could actually have a single code address that corresponds to multiple source locations if the <a href="https://devblogs.microsoft.com/oldnewthing/20050322-00/?p=36113">linker/optimizer combines multiple functions together</a> because they had the same generated code. In most cases, there's not a good way to show this in an interactive debugger, and WinDbg in particular handles this by just showing a single source resolution. That can be particularly confusing when you set a source level breakpoint in one place but it gets displayed on a completely different source line! Especially when the relationship between those two functions isn't clear. When I worked on WinDbg, I always wanted to do better for these cases by trying to use more context clues to figure out a more specific resolution (e.g. when resolving an address in a call stack, determine what function was originally called and use that version), but in reality the functions that get folded together are <i>usually</i> fairly trivial.
  </li>
  <li id="source-file-info">
  WinDbg will keep track of what module each source file was loaded from, which allows it to search only that module for breakpoint resolution. A source file that was loaded from an address (like one that you navigate to from a call stack) will have this module assocation, but a file opened directly (e.g. from the "open source file" dialog) will not. This can cause *drastically* different performance for breakpoint resolution because setting a breakpoint on a source file without an associated module will end up needing to search through every single loaded module to find a breakpoint resolution! We could implement support for optionally searching through all modules, but that's easy enough to add later.
  </li>
  </ol>
</footer>