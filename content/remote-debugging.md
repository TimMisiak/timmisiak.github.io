---
title: "Remote debugging"
date: 2022-08-05T07:00:00-07:00
draft: false
---


A key feature of WinDbg and NTSD is the ability to debug a target "remotely" from a separate computer. For kernel debugging, this is often the only way to debug, since the entire OS is "frozen" when broken into a kernel debugger. Remote debugging is also available for usermode debugging, and is often just as useful. Sometimes it's useful because the target that you are testing on is different from the one you are using for development. Other times, it's useful because you want to debug a critical usermode process such as part of the graphics stack. Whatever the reason you have for debugging remotely, this post will talk about how to get remote debugging set up without falling into some common pitfalls.

# Terms: Target, host, and remote

Just to define some terms and avoid confusion later: When we talk about remote debugging, we talk in terms of "targets" and "hosts". The "target" is the computer where the target process or OS is running. The "host" is where you are controlling the debugging session from (usually from WinDbg). We also sometimes talk about a "remote", which is just short for "remote debugging server".

# NTSD or DbgSrv?

Before you start setting up remote debugging, you need to decide what type of remote debugging server to use. The most common type of remote debugging is done using NTSD/CDB running on the target OS. While you can also use WinDbg as your remote debugging server, the console debuggers tend to work in more cases, and are easy to set up as remote debugging servers. You may have also heard of "DbgSrv" as a debugging server (sometimes called a [Process Server](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/process-servers--user-mode-)). While they might seem similar at first glance, using NTSD as a remote debugging server is very different from using DbgSrv. When using NTSD as your debugging server, all of the "brains" of the debugger are on the *target* side. When using DbgSrv, all of the "brains" of the debugger are on the *host* side. That's important when thinking about where your extensions and your symbols are. If you build a binary locally on your host machine and then transfer the binaries over to the target machine, you **also** need to transfer the symbol (PDB) files to the target machine if you are using NTSD remote debugging. In contrast, when debugging remotely using DbgSrv, the remote target is a very simple "stub" that just provides the minimum set of services needed to debug remotely, like reading and writing memory or registers, as well as relaying all of the debugging events like exceptions and module loads. The main advantage is that all of the symbols and extensions stay on the host, and a minimal number of files need to be moved to the target. A disadvantage is that the debugging session will often be slower (especially for high latency connections).

You can read more about the differences on [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/choosing-the-best-remote-debugging-method).

In summary:

## NTSD remote debugging

* All of the "brains" are on the **target**
* Symbols and extensions are on the **target**
* Source files are on the **host**
* Fastest way to do remote debugging on high latency connections

## DbgSrv remote debugging

* All of the "brains" are on the **host**
* Symbols and extensions are on the **host**
* Source files are on the **host**
* Debugging sessions can be slower
* A small set of files are needed on the target (you can get by with just dbgsrv.exe and dbgeng.dll)

# Setting up an NTSD remote

Setting up a basic NTSD remote is easy. Imagine you are debugging MyTarget.exe on your remote target computer. You can start an ntsd debugging session with:

```
ntsd.exe MyTarget.exe
```

Then, once the target breaks in, you can run:

```
0:000> .server tcp:port=1234
Server started.  Client can connect with any of these command lines
0: <debugger> -remote tcp:Port=1234,Server=mytarget
```

The ".server" command sets up a debugging server over TCP using port 1234. To connect to this debugging server from another machine using WinDbg, you can paste this command line to run:

```
windbgx.exe -remote tcp:Port=1234,Server=mytarget
```

Running this will immediately connect to the remote debugging session. You'll have a shared session between the two debuggers and can type in commands on either side and see the same results on both sides of the connection.

Note that you can use any debugger with the "-remote" command, including NTSD, CDB, KD, WinDbgX (WinDbg Preview), or WinDbg "classic". The core debugging engine is the same for all of these, and they all support the "-remote" command line option.

While you can always use a ".server" command to start a debugging server on your target, you can also do this directly from the command line with the "-server" command line argument:

```
ntsd.exe -server tcp:port=1234 MyTarget.exe
```

This can be convenient if you are starting your debugging server from a script or other automated method.

Note that there are other transports besides "tcp", such as "npipe" which uses named pipes. You can read about the other transports supported on [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/-server--create-debugging-server-).

# Setting up a DbgSrv remote

To set up a DbgSrv remote, you start by setting up DbgSrv on your **target** with a connection string:

```
dbgsrv.exe -t tcp:port=1234
```

In this case, dbgsrv will start running but will display no output if everything went well. If something went wrong, it will display a message box with a cryptic error. Usually this means the port was already in use.

To connect to the dbgsrv remote from your **host** machine, you will use the "-premote" command line argument instead of the "-remote" argument. Since the dbgsrv remote does not already have an active debugging session, you also need to tell it what debugging session to start. For instance, you can now run this on your **host** machine:

```
ntsd.exe -premote tcp:port=1234,server=mytarget MyTarget.exe
```

This will connect to the dbgsrv process server and start a process called "MyTarget.exe".

If you are using WinDbg, you can also connect to a process server without starting a debugging server. For instance:

```
WinDbgX -premote tcp:port=1234,server=mytarget
```

This will connect WinDbg to the process server and let you use the UI to launch or attach to a process.

You can read more about connecting to process servers on [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/activating-a-process-server)

Note that dbgsrv.exe will continue running on the remote system until it is killed. It has no UI, so you need to end the process manually when you are done.

## The "Implicit command line"

There is one other way to use dbgsrv that isn't widely used (to my knowledge) but can sometimes be really useful. That's using the "implicit command line" parameter to dbgsrv. To use this mode, you can run on the **target** machine:

```
dbgsrv -t tcp:port=1234 -pc MyTarget.exe
```

The "-pc" command line parameter is saved by dbgsrv as the "implicit command line". To tell dbgsrv to use the implicit command line, you can run ntsd (or windbg) with a command line like this on the **host** machine:

```
ntsd -premote tcp:port=1234,server=mytarget -cimp
```

This will connect to dbgsrv running on your target machine and launch "MyTarget.exe", since this was saved as the implicit command line. This can be useful, especially combined with the "-sifeo" parameter to dbgsrv, since it allows it to be registered as the debugger via "IFEO" (Image File Execution Options). Using IFEO has its own set of pitfalls, so I'll leave the details to another blog post.