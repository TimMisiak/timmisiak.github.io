---
title: "Symbol and Binary Indexing"
date: 2022-08-10T08:30:09-07:00
draft: false
---

Symbol indexing is one of those features of WinDbg that can make things "just work" in a way that seems like magic. But it can also be the most painful things when it goes wrong.

# Why should I index symbols and binaries?

Most of us have tried to debug without symbols at one point, and it can quickly become an exercise in frustration. It's much more productive to debug an executable where you have symbols because it gives you function names, variable names, type definitions, and source files. Since most Windows applications don't ship with private symbols (with some notable [exceptions](https://twitter.com/timmisiak/status/1533483161954308096)), we need some way to get the symbols for a binary. That's where symbol indexing and symbol servers come into play. Every binary built with symbols comes with a section of data called the "debug directory" which contains information about its symbols. Windows debuggers (including Visual Studio and WinDbg) can use the debug directory to download symbols from a shared symbol server. Usually symbols are published to this server as part of the build process for a product. Microsoft hosts a public symbol server for most of its products at ```https://msdl.microsoft.com/download/symbols```.

While it might seem obvious that symbols are required to be productive in the debugger, it is less obvious that binaries are also needed and should be indexed on the symbol server. It's easy to miss that the binaries are needed, since private builds and live usermode debugging sessions usually have the binaries available local in places that the debugger can easily find or read from memory. On the other hand, usermode minidumps do not always contain the full memory for loaded modules to save space in the dump file, and kernel dumps may not contain the full memory of loaded modules if parts of them are paged out when the dump is captured. When the full binary is not available, the debugger can use the information in the header to download the binary from the symbol server and "map" it in so that the data is available just as if it were captured in the dump. The contents of the binaries are critical for a number of things, but the most important part is for stack walking. The binaries contain information for the OS to perform stack unwinds and for debuggers to walk the stack. If you index your symbols but not your binaries, things may seem to work at first if you debug a live process or a dump that was captured from the same machine as the debugger, but as soon as you start debugging a crash dump that came from somewhere else, however, you'll start to see problems in the stack unwind and elsewhere.

# How can I index symbols and binaries?

Indexing symbols and binaries require having a server to store the archive of binaries. If you are using Azure DevOps, the easiest way to do this is with the [PublishSymbols task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/index-sources-publish-symbols?view=azure-devops). Azure DevOps has a symbol server that you can reference using:

```
.sympath+ https://artifacts.dev.azure.com/<ORGANIZATION_NAME>/_apis/symbol/symsrv
```

If you are not using Azure DevOps, there are a number of other solutions that you can use. A symbol server can be hosted on an SMB share or an HTTPS server. All that is needed is a directory structure of files that follows a convention that allows the debugger to find the files. The structure of the files is ```<symbolServerRoot>/<fileName>/<index>/<fileName>```. The file structure is the same whether you are using an SMB or an HTTPS server. You can read more about this on [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/symbol-store-folder-tree). The [SymStore](https://docs.microsoft.com/en-us/windows/win32/debug/using-symstore) tool can be used to more easily create the directory structure for a symbol server, although this solution does not scale well to large numbers of symbols, as it uses flat text files for tracking the symbol and binary files.

# How does WinDbg find binaries and symbols on a symbol server?

The above should be enough information to get you started on publishing and consuming symbol servers, but to really understand what's going on, we'll need to look at how a debugger determines the index for a file when talking to a symbol server. How does the debugger know which binary to download for a given minidump? And for that binary, how does it know which symbols to download?

To start with, we'll capture a small minidump that doesn't include the full binary data. You can do that directly from the debugger using:

```
.dump /m smallDump.dmp
```

If we open that dump and use the ```.dumpdebug``` command, it will spew a diagnostic log of everything contained in the dump. One of the streams of the minidump file is the ```ModuleListStream```:

```
0:000> .dumpdebug
...
Stream 1: type ModuleListStream (4), size 00000658, RVA 000007F0
  15 modules
  RVA 00000D70, 00007ffb`2c100000 - 00007ffb`2c454000: 'C:\Windows\System32\combase.dll', DllCharacteristics: 4160, Timestamp: afbf9ef6, CheckSum: 3650a6
```

There are three things we can determine from this that are used to find the original file:

* The filename (combase.dll)
* The timestamp (afbf9ef6)
* The size (354000)

For binaries, the index is the timestamp and size concatenated together. As mentioned above, the structure of the URL is ```<symbolServerRoot>/<fileName>/<index>/<fileName>```. Assuming we are downloading this file from the Microsoft public symbol server, that gives us a URL of:

```
https://msdl.microsoft.com/download/symbols/combase.dll/AFBF9EF6354000/combase.dll
```

You can test this by pasting the URL into a browser or using PowerShell to download the file:

```
Invoke-WebRequest -Uri "https://msdl.microsoft.com/download/symbols/combase.dll/AFBF9EF6354000/combase.dll" -OutFile combase.dll
```

We can look at the noisy symbol downloading output from WinDbg to see that this matches what it uses for downloading the file:

```
0:000> !sym noisy
noisy mode - symbol prompts on
0:000> .reload -f combase.dll
SYMSRV:  BYINDEX: 0x5
         C:\ProgramData\Dbg\sym
         combase.dll
         AFBF9EF6354000
SYMSRV:  UNC: C:\ProgramData\Dbg\sym\combase.dll\AFBF9EF6354000\combase.dll - path not found
SYMSRV:  UNC: C:\ProgramData\Dbg\sym\combase.dll\AFBF9EF6354000\combase.dl_ - path not found
SYMSRV:  UNC: C:\ProgramData\Dbg\sym\combase.dll\AFBF9EF6354000\file.ptr - path not found
SYMSRV:  RESULT: 0x80070003
SYMSRV:  BYINDEX: 0x6
         C:\ProgramData\Dbg\sym*https://msdl.microsoft.com/download/symbols
         combase.dll
         AFBF9EF6354000
SYMSRV:  UNC: C:\ProgramData\Dbg\sym\combase.dll\AFBF9EF6354000\combase.dll - path not found
SYMSRV:  UNC: C:\ProgramData\Dbg\sym\combase.dll\AFBF9EF6354000\combase.dl_ - path not found
SYMSRV:  UNC: C:\ProgramData\Dbg\sym\combase.dll\AFBF9EF6354000\file.ptr - path not found
SYMSRV:  HTTPGET: /download/symbols/index2.txt
SYMSRV:  HttpQueryInfo: 80190194 - HTTP_STATUS_NOT_FOUND
SYMSRV:  HTTPGET: /download/symbols/combase.dll/AFBF9EF6354000/combase.dll
SYMSRV:  HttpQueryInfo: 801900c8 - HTTP_STATUS_OK
```

Now that we have the image file, we can use it to find the symbol file. To do this we need to look at the debug directories for the binary. One way is with ```link.exe``` from the compiler tools:


```
C:\>link /dump /headers combase.dll
Microsoft (R) COFF/PE Dumper Version 14.32.31332.0
Copyright (C) Microsoft Corporation.  All rights reserved.
...
  Debug Directories

        Time Type        Size      RVA  Pointer
    -------- ------- -------- -------- --------
    AFBF9EF6 cv            24 002C9910   2C8310    Format: RSDS, {42AC0098-ADB5-E796-8C0E-161D2814780E}, 1, combase.pdb
    AFBF9EF6 coffgrp      8D0 002C9934   2C8334    50475500 (PGU)
    AFBF9EF6 repro         24 002CA204   2C8C04    98 00 AC 42 B5 AD 96 E7 8C 0E 16 1D 28 14 78 0E DA C5 A7 A4 CE 3F B7 F3 19 C0 EE 8F F6 9E BF AF
    AFBF9EF6 dllchar        4 002CA228   2C8C28
```

There are a few debug directories in this binary, but the one used for looking up the PDB file is the "cv" record, which contains the "RSDS" signature, a GUID, an "age" field, and the filename for the PDB. If you concatenate the GUID (with only the hex digits) and the age (without any leading zeroes), you get the index for the PDB. Combined with the filename we end up with a URL of:

```
https://msdl.microsoft.com/download/symbols/combase.pdb/42AC0098ADB5E7968C0E161D2814780E1/combase.pdb
```

Download this like before and we end up with the symbols for this version of combase.dll!

# Miscellaneous notes

A few other notes about this that didn't fit anywhere else:

* Besides using link.exe for dumping the debug directory, you can also use the ```!dh``` command in the debugger, although the output is slightly less friendly.
* The "index2.txt" file can be put at the root of a symbol store to change the file structure slightly. This is called a "Two-Tier Structure", and you can read more about this on [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/symbol-store-folder-tree)
* Files on a symbol store can also be compressed as CAB or ZIP files. If compressed, the files should be named with the last character as an underscore. For instance, ```combase.dll\AFBF9EF6354000\combase.dl_```. 
* The "timestamp" field of a binary may not be a timestamp if the debug directory contains a ```repro``` type entry. The debugger generally doesn't care about that when downloading binaries though, since it still works the same way.
* The debugger doesn't always "map" images in when debugging kernel dumps where it would be useful. That's been on my backlog for a while and is something I will be fixing in the next few months.