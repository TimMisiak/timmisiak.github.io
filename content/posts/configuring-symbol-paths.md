---
title: "Configuring Symbol Paths for Tools"
date: 2022-08-11T08:30:09-07:00
draft: true
---

In my last post, I talked about how [symbol and binary indexing works](symbol-indexing.md), and how WinDbg uses those files. WinDbg isn't the only tool that cares about symbols though. There are many tools that use symbols for various purposes. Some of them use symbols for stack walks, others. A few tools that use symbols on windows:

* Monitoring and similar utilities
  * [Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon)
  * [Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer)
  * [Process Hacker](https://processhacker.sourceforge.io/)
* Debuggers/disassemblers
  * [Visual Studio](https://docs.microsoft.com/en-us/visualstudio/debugger/specify-symbol-dot-pdb-and-source-files-in-the-visual-studio-debugger?view=vs-2022)
  * [IDA](https://hex-rays.com/ida-pro/)
  * [x64dbg](https://help.x64dbg.com/en/latest/commands/analysis/symload.html)
  * [ollydbg](https://www.ollydbg.de/whatsnew.htm)
  * [ILSpy](https://github.com/icsharpcode/ILSpy)
  * [Binary Ninja](https://binary.ninja/)
  * [Ghidra](https://ghidra-sre.org/)
* Profilers
  * [Windows Performance Analyzer](https://docs.microsoft.com/en-us/windows-hardware/test/wpt/load-symbols-or-configure-symbol-paths)
  * [PerfView](https://github.com/microsoft/perfview)
* Other
  * [SizeBench](https://aka.ms/SizeBench)
  * [DTrace](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/dtrace)
  * [Live++](https://liveplusplus.tech/)

(This list is not meant to be exhaustive, and was crowdsourced from folks who follow me on Twitter)