---
title: "Writing a Debugger From Scratch - DbgRs Part 4 - Exports and Private Symbols"
date: 2023-05-29T15:28:40-07:00
draft: false
---

(New to this series? Consider starting from [part 1](/posts/writing-a-debugger-from-scratch-part-1))

At the end of the [last post](/posts/writing-a-debugger-from-scratch-part-3), we had the ability to read memory out of the target process, but we still had very little in the way of interpreting the data in that process. That changes in this post, where we will start grabbing useful information out of the target, including modules, exports, and even private symbols.

The code for this post is in the [part4 branch on github](https://github.com/TimMisiak/dbgrs/tree/part4). If you see any mistakes or ways to improve the code, feel free to [create issues](https://github.com/TimMisiak/dbgrs/issues) in the GitHub repo or submit a PR.

# Where are the modules?

Before we can start reading the exports or private symbols of a module, we need to know what modules are loaded and where they are loaded. We already have a handler for LOAD_DLL_DEBUG_EVENT that comes from WaitForDebugEventEx, which includes both the address and the name. We'll keep a list of all loaded modules so we can easily find which module is associated with an address, and then consult the information for that module to map the address to a name. Since a loaded module instance is associated with a process, we'll create a Process struct to keep track of all of the loaded modules.

```rust
pub struct Process {
    module_list: std::vec::Vec<Module>,
}
```

We'll make a function for adding a module to the loaded list, and defer to the Module object for doing the parsing.

```rust
impl Process {
    pub fn new() -> Process {
        Process { module_list: Vec::new() }
    }

    pub fn add_module(&mut self, address: u64, name: Option<String>, memory_source: &dyn MemorySource) ->
            Result<&Module, &'static str> {
        let module = Module::from_memory_view(address, name, memory_source)?;
        self.module_list.push(module);
        Ok(self.module_list.last().unwrap())
    }
```

<div class="pe32">
{{% figure src="/pe32.svg#pe32" caption="PE32 structure" %}}
<small>Originally from [Wikipedia](https://en.wikipedia.org/wiki/File:Portable_Executable_32_bit_Structure_in_SVG_fixed.svg). Modified by cropping to content. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.en)</small>
</div>

We're passing in all the information that will be needed to understand the module, which is essentially just the address of the module and the memory source to read the data from. We'll also pass in the module name given from the load event if it's available. If it's not available, we'll get the name from the loaded image in the <a aria-describedby="footnote-label" href="#process-name">target process</a>.

Now we're ready to start parsing the information in the module. Windows modules are generally "PE files", or "Portable Executable". The PE format is [fairly well documented](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format), but it will be easier to understand if you look at a diagram of the structure. The diagram to the right is the 32-bit PE format (PE32), but this is essentially the same as the 64-bit format (PE32+) with a <a aria-describedby="footnote-label" href="#pe32-64">few small differences</a>.

The first few headers are there for legacy reasons. The ```IMAGE_DOS_HEADER``` exists so that executables would be recognized by DOS. At the end of this header is a field called ```e_lfanew``` which is the offset to the "real" header. The "DOS stub" that is after this header was used so that a Windows executable run under DOS would display a message saying "This program cannot be run in DOS mode". That stub weirdly persists to this day, and even exists in DLLs where it's hard to imagine anyone would try to run them directly.

We can ignore everything else in the DOS header and go straight to reading the "real" header, which is the Windows header. This part is a few structs that are laid out consecutively in memory, and the "optional" header <a aria-describedby="footnote-label" href="#optional-header">isn't really optional for image files</a>, so we can read them all at once as a single structure read. Conveniently, the [```IMAGE_NT_HEADERS64```](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-image_nt_headers64) struct contains the ```IMAGE_FILE_HEADER``` (called "COFF Header" in the diagram) and the [```IMAGE_OPTIONAL_HEADER64```](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-image_optional_header64) (called the "optional header" in the diagram). A few fields are interesting here, such as the <a aria-describedby="footnote-label" href="#size-of-image">SizeOfImage field that tells us how large the image is</a>.

There are two tables in the header that describe the rest of the data in the file. The first is the "section table" which describes how each part of the image file should be loaded into memory, including information like what address the section should be loaded at and the size of the section. It also contains flags marking whether the section should be readable, writable, and/or executable. For instance, any sections containing code should be marked as readable and executable, while sections containing globals should be readable and writable. Each section also has a name. The names can be nearly anything, although there are some [reserved names with special meaning](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#special-sections), such as ".text" for the code section or ".data" for initialized data. Note that most "pointers" to data in the file are described as "RVAs" or "[Relative Virtual Addresses](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#general-concepts)". This means that the address are relative to the start of the image *after* it's loaded into memory. This is important to know when interpreting a file on disk, but since we are reading from images that are already loaded into memory with sections at the expected addresses, we can ignore the section table for the moment. Each time we see an RVA, we can just add it to the module base address and we'll get the correct location.

The second table is the "directory table", and is the more interesting one for us right now. The size of the table is described by the NumberOfRvaAndSizes field of the optional header and is typically 16 entries. Each entry of the table just has an RVA and length describing the location of the data. The role of each entry is determined by the index. For instance, the export table is always at index 0 and has a corresponding constant ```IMAGE_DIRECTORY_ENTRY_EXPORT```. You can see this as the first entry in the diagram on the right, labled "ExportTable".

Although this has been a long explanation describing the structure of the headers, all we need to do so far is to read the DOS header, find the offset to the Windows header, and then read the Windows header so that we can get to the data directories.

```rust
impl Module {
    pub fn from_memory_view(module_address: u64, module_name: Option<String>, memory_source: &dyn MemorySource) -> 
            Result<Module, &'static str> {

        let dos_header: IMAGE_DOS_HEADER = memory::read_memory_data(memory_source, module_address)?;
        // NOTE: Do we trust that the headers are accurate, even if it means we could read outside the bounds of the
        //       module? For this debugger, we'll trust the data, but a real debugger should do sanity checks and 
        //       report discrepancies to the user in some way.
        let pe_header_addr = module_address + dos_header.e_lfanew as u64;

        // NOTE: This should be IMAGE_NT_HEADERS32 on x86 processes
        let pe_header: IMAGE_NT_HEADERS64 = memory::read_memory_data(memory_source, pe_header_addr)?;
```

## The export table

Now that we have read the data directories, we can find the export table. The export table of a DLL has a list of names and the corresponding addresses for those names. Much like static linking, where function names are resolved to addresses at compile time, the export table allows function names to be resolved dynamically at runtime, thus the name "Dynamic Link Library". This means that even for a DLL with no private symbols, we will still know the address of any <a aria-describedby="footnote-label" href="#data-exports">functions that are exported</a>. While exports generally have names, it's also possible for exports to be referenced by "ordinal", which is just an index into the export table.

The exports are described by the [```IMAGE_EXPORT_DIRECTORY```](https://microsoft.github.io/windows-docs-rs/doc/windows/Win32/System/SystemServices/struct.IMAGE_EXPORT_DIRECTORY.html) structure. We can read this using the address from the directory table.

```rust
fn read_exports(pe_header: &IMAGE_NT_HEADERS64, module_address: u64, memory_source: &dyn MemorySource) ->
        Result<(Vec::<Export>, Option<String>), &'static str> {
    let mut exports = Vec::<Export>::new();
    let mut module_name: Option<String> = None;
    let export_table_info = pe_header.OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT.0 as usize];
    if export_table_info.VirtualAddress != 0 {
        let export_table_addr = module_address + export_table_info.VirtualAddress as u64;
        let export_table_end = export_table_addr + export_table_info.Size as u64;
        let export_directory: IMAGE_EXPORT_DIRECTORY = memory::read_memory_data(memory_source, export_table_addr)?;
```

Besides the exports themselves, this structure may also contain the name of the module. We'll use that as another source of information if we don't already have the module name:

```rust
        // This is a fallback that lets us find a name if none was available.
        if export_directory.Name != 0 {
            let name_addr = module_address + export_directory.Name as u64;
            module_name = Some(memory::read_memory_string(memory_source, name_addr, 512, false)?);
        }
```

The exports are described as two logical tables. The first is the addresses of the functions themselves as 32-bit RVAs and is described by the ```AddressOfFunctions``` and ```NumberOfFunctions``` fields.

```rust
        let address_table_address = module_address + export_directory.AddressOfFunctions as u64;
        let address_table = memory::read_memory_full_array::<u32>(memory_source,
                                                                  address_table_address,
                                                                  export_directory.NumberOfFunctions as usize)?;
```

The second table is the "name table" which maps ordinals to names. It is a single logical table stored in the file as two parallel arrays, one containing the ordinals and one containing <a aria-describedby="footnote-label" href="#name-ordinal-table">RVAs to the names</a>. The ```NumberOfNames``` field describes the number of entries in both arrays, and the ```AddressOfNameOrdinals``` and ```AddressOfNames``` are the RVAs to each array.

```rust
        // We'll read the name table first, which is essentially a list of (ordinal, name) pairs that give names 
        // to some or all of the exports. The table is stored as parallel arrays of orindals and name pointers
        let ordinal_array_address = module_address + export_directory.AddressOfNameOrdinals as u64;
        let ordinal_array = memory::read_memory_full_array::<u16>(memory_source,
                                                                  ordinal_array_address,
                                                                  export_directory.NumberOfNames as usize)?;
        let name_array_address = module_address + export_directory.AddressOfNames as u64;
        let name_array = memory::read_memory_full_array::<u32>(memory_source,
                                                               name_array_address,
                                                               export_directory.NumberOfNames as usize)?;
```

As I mentioned previously, not all exports have names. These can be referenced by "ordinal", which is an index into the table. The ```Base``` field of the export table describes the ordinal of the first element of the table. So if this value is 100, then finding ordinal 105 in the export would be at index number 5 in the address table.

The last piece of information needed for parsing this table is that some entries in the address table are "[forwarders](https://devblogs.microsoft.com/oldnewthing/20060719-24/?p=30473)". These entries describe exports that are forwarded to another DLL by name, and will be resolved to the export in a target DLL. Any address that points to data inside the export section is forwarder, and is pointing to a string in the form of "MODULENAME.EXPORTNAME", where MODULENAME is a dll name and EXPORTNAME is either a function name like "NTDLL.RtlAcquireSRWLockExclusive" or an ordinal with a '#' prefix, such as "NTDLL.#24". If the target address is outside the export section, it is a normal export.

We'll build up a table of all exports so that we can quickly look up addresses. To start, we'll iterate over the address table.

```rust
        for (unbiased_ordinal, function_address) in address_table.iter().enumerate() {
            let ordinal = export_directory.Base + unbiased_ordinal as u32;
            let target_address = module_address + *function_address as u64;
```

Then we'll check if the name table has an entry for this ordinal, and read it from the target process if it exists.

```rust
            let name_index = ordinal_array.iter().position(|&o| o == unbiased_ordinal as u16);
            let export_name = match name_index {
                None => None,
                Some(idx) => {
                    let name_address = module_address + name_array[idx] as u64;
                    Some(memory::read_memory_string(memory_source, name_address, 4096, false)?)
                }
            };

```

Finally, we'll check if this export is a normal export or if it's a forwarder. We'll add this to an "exports" list, and store this as part of the Module struct.

```rust
            // An address that falls inside the export directory is actually a forwarder
            if target_address >= export_table_addr && target_address < export_table_end {
                // I don't know that there actually is a max size for a forwader name, but 4K is probably reasonable.
                let forwarding_name = memory::read_memory_string(memory_source, target_address, 4096, false)?;
                exports.push(Export {name: export_name, ordinal, target: ExportTarget::Forwarder(forwarding_name)});                    
            } else {
                exports.push(Export{name: export_name, ordinal, target: ExportTarget::RVA(target_address)});
            }
        }
```

## Private symbols

There is another source of names we can use, and that's from private symbols. To find the corresponding private symbols for a binary, we need to look at the debug directory, which is at the ```IMAGE_DIRECTORY_ENTRY_DEBUG``` index of the directory table. The debug directory is itself a table of ```IMAGE_DEBUG_DIRECTORY``` structs, and we can determine how many are available based on the size of the debug directory. There are a few different types of debug info that can be stored here, but the one we care about for now is the "codeview" entry.

```rust
    let debug_table_info = pe_header.OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_DEBUG.0 as usize];
    if debug_table_info.VirtualAddress != 0 {
        let dir_size = std::mem::size_of::<IMAGE_DEBUG_DIRECTORY>() as u64;
        // We'll arbitrarily limit to 20 entries to keep it sane.
        let count: u64 = std::cmp::min(debug_table_info.Size as u64 / dir_size, 20);
        for dir_index in 0..count {
            let debug_directory_address = module_address + (debug_table_info.VirtualAddress as u64) + (dir_index * dir_size);
            let debug_directory: IMAGE_DEBUG_DIRECTORY = memory::read_memory_data(memory_source, debug_directory_address)?;
            if debug_directory.Type == IMAGE_DEBUG_TYPE_CODEVIEW {
```

The CodeView entry has a signature, a guid, an "age", and a name.

```rust
#[repr(C)]
pub struct PdbInfo {
    pub signature: u32,
    pub guid: windows::core::GUID,
    pub age: u32,
    // Null terminated name goes after the end
}
```

We can use this struct to read the data from the table.

```rust
        let pdb_info_address = debug_directory.AddressOfRawData as u64 + module_address;
        pdb_info = Some(memory::read_memory_data(memory_source, pdb_info_address)?);
        // We could check that pdb_info.signature is RSDS here.
        let pdb_name_address = pdb_info_address + std::mem::size_of::<PdbInfo>() as u64;
        pdb_name = Some(memory::read_memory_string(memory_source, pdb_name_address, 260, false)?);
```

We could use the fields of the struct to download the symbols from a symbol server or symbol cache, but we'll leave that functionality for later. For now we'll assume that the name points to an absolute path on disk and use the [pdb crate](https://crates.io/crates/pdb) to load the data.

```rust
        let pdb_file = File::open(pdb_name.as_ref().unwrap());
        if let Ok(pdb_file) = pdb_file {
            let pdb_data = PDB::open(pdb_file);
            if let Ok(pdb_data) = pdb_data {
                pdb = Some(pdb_data);
            }
        }
```

## Name resolution

Now that we have exports and private symbols, we can start to resolve addresses to names as we are debugging. We'll create a new function called ```resolve_address_to_name``` that takes a process and an address and returns a symbol name with offset if it can be found.

```rust
pub fn resolve_address_to_name(address: u64, process: &mut Process) -> Option<String> {
    let module = match process.get_containing_module_mut(address) {
        Some(module) => module,
        None => return None
    };
```

For now we'll just do a linear search over the exports and find the closest export that comes before the address we are mapping.

```rust
    for export in module.exports.iter() {
        if let ExportTarget::RVA(export_addr) = export.target {
            if export_addr <= address {
                if closest.is_none() || closest_addr < export_addr {
                    closest = AddressMatch::Export(export);
                    closest_addr = export_addr;
                }
            }
        };
    }
```

We'll do the same thing for the symbols in the pdb. Once we find the closest match, we'll return a string that represents the symbol and add a "+offset" suffix if the address isn't an exact match.

```rust
    if let AddressMatch::Export(closest) = closest {
        let offset = address - closest_addr;
        let sym_with_offset = if offset == 0 {
            format!("{}!{}", &module.name, closest.to_string())
        } else {
            format!("{}!{}+0x{:X}", &module.name, closest.to_string(), offset)
        };
        return Some(sym_with_offset)
    }
```

Now that we can resolve an address to a string, we can resolve the instruction pointer to a name each time the debugger breaks in.

```rust
    if let Some(sym) = name_resolution::resolve_address_to_name(ctx.context.Rip, &mut process) {
        println!("[{:X}] {}", debug_event.dwThreadId, sym);
    } else {
        println!("[{:X}] {:#018x}", debug_event.dwThreadId, ctx.context.Rip);
    }
```

And to let us look up any address we want, we'll add a command called ```ln``` to let us look up a given address.

```rust
    CommandExpr::ListNearest(_, expr) => {
        let val = eval::evaluate_expression(*expr);
        if let Some(sym) = name_resolution::resolve_address_to_name(val, &mut process) {
            println!("{}", sym);
        } else {
            println!("No symbol found");
        }
    }
```

## Testing it out

Let's see it all in action!

```
Command line was: '"C:\git\HelloWorld\hello.exe" '
CreateProcess
LoadDll: 7FF7D9D20000   hello.exe
[2E86C] 0x00007fff20eea9d0
> g
LoadDll: 7FFF20E90000   ntdll.dll
[2E86C] ntdll.dll!RtlUserThreadStart
> ln 0x00007ff7d9d27100  
hello.exe!main
[2E86C] ntdll.dll!RtlUserThreadStart
> g
LoadDll: 7FFF1FCA0000   C:\Windows\System32\KERNEL32.DLL
[2E86C] ntdll.dll!NtMapViewOfSection+0x14
>
```

It works! You'll note that on the very first break, we have a bare address that can't be resolved. That's because the LoadDll notification hasn't been received yet for ntdll, so we don't have enough information to resolve the address. On the second debug event, we get information about ntdll, which lets us resolve the address to ```ntdll!RtlUserThreadStart```. Finally, we try to resolve the address for a private symbol, which you can see resolves to ```hello!main```.

This post is a big step for dbgrs, since we're finally interpreting things in a more human friendly way. There's still not quite enough functionality here to do anything useful though, since we can't really control the state of the target in a way that lets us do any interesting debugging. I think the lack of breakpoints is the biggest weakness. Now that we have some basic symbol resolution, we have what we need to start building that functionality, and that's where we'll work on in the next post.

Hope you found this post interesting and informative! Have a question or suggestion? Let me know! You can find me on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim).

<footer>
  <h2 id="footnote-label">Footnotes</h2>
  <ol>
  <li id="process-name">There are a bunch of different sources of information you could use as the name of a module. You could use the name from the debug event (if present), the name from the file handle from the debug event, the name from the export table (if present), the name from the resource table (if present), or even derive the name from the debug directory. Maybe even some more sources that I'm forgetting. The tricky part is that there's no reason why all four of these need to match, and they are often different! Add this to the fact that you could have two instances of the same module loaded, and the concept of a "module name" gets very complicated. We'll take a similar approach to what I believe WinDbg does, which is to prioritize the name from the module load event, and fall back to other sources if needed. 
  </li>
  <li id="pe32-64">Most fields are actually the same. The main differences are a few fields that are memory sizes or addresses, such as the ImageBase, and size of stack/heap reserve and commit. Many of the fields are RVAs, which are relative to the image base, so 32-bit offsets are generally fine.
  </li>
  <li id="optional-header">It's called "optional" because much of this format is shared with object files and <a href="https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#optional-header-image-only">object files do not have this header</a>.
  </li>
  <li id="size-of-image">We could also get the size of image from the memory allocation, looking for the consecutive set of MEM_IMAGE allocations using VirtualQueryEx. Like so many other things about debugging a live process, the information can come from various sources and you need to pick which one to trust. If I were writing a real debugger and not one for educational purposes, I would consider reading all of the sources and reporting to the user whenever they don't match. Having multiple sources of information to cross check can be useful for cases such as corrupt files or hardware errors.
  </li>
  <li id="data-exports">Strictly speaking, exports can be used for both functions and data. In practice, using exports for data is somewhat rare although it does happen. It's generally not a great design to have shared writable data exported, but I've seen it used to describe metadata about a module. <a href="https://developer.download.nvidia.com/devzone/devcenter/gamegraphics/files/OptimusRenderingPolicies.pdf">Nvidia</a> and <a href="https://gpuopen.com/learn/amdpowerxpressrequesthighperformance/">AMD</a> both have some special exports you can use to tell the drivers what performance mode you want. (Thanks <a href="https://twitter.com/molecularmusing/status/1662941499770695681">Stefan</a>!)
  </li>
  <li id="name-ordinal-table">The tables are stored separately because the ordinals are 16 bit integers and the name RVAs are 32 bit integers. So instead of having an array of 48 bit elements, there is an array of 16 bit elements and an array of 32 bit elements, to keep everything aligned and avoid unaligned accesses (which can be slower on some architectures).
  </li>
  </ol>
</footer>