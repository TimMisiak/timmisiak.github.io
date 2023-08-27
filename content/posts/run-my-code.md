---
title: "Run My Code! (code injection on Windows)"
date: 2023-08-27T15:13:28-07:00
draft: false
---

The first time I realized it was possible to get a process to run some extra little code I had written, it felt like the ultimate cheat code. My first attempt was a little patch for Civilization 2 to fix some high CPU usage. Then I discovered that you could inject code at *run-time*. And when I discovered the ability to *change* how OS functions worked, it started to feel like I could do anything!

Code injection gets used for many purposes, sometimes legitimate, sometimes nefarious. But how do you actually go about injecting code? When most people hear code injection, they think of things like buffer overflow attacks and return oriented programming. These rely on discovering vulnerabilities in some target program. What I often find more interesting are the ways that you can inject code into *any* process, regardless of security vulnerabilities.

On Windows, there are a number ways to get code running in a another process, but they all fall into three main categories. Static code injection (patching), DLL injection, and dynamic code injection.

# Static code injection (Patching)

Static code injection is one obvious way to inject code into a process. Just change the executable while it's still on the disk, before it gets a chance to load. If you just need to do a little bit of modification, like patching in a call to a ```Sleep``` function in some tight loop, you might even be able to do it manually by using WinDbg to do some patching at runtime, and then figuring out the bytes that need to change in the file.

This method has some drawbacks. First, you have to actually modify the file on disk, which could be more destructive than you want (e.g. it affects every run of the program, instead of just some instances). Changing the files on disk is also problematic for signed binaries where any changes would invalidate the code signing. And figuring out how to patch binaries in some general way that works for several different versions of a binary is not an easy task.

# DLL injection

DLL injection is the most widely used method of injecting code into a process, especially for *legitimate* code injection. It's popular because the technique is straightforward, and there are a large number of documented mechanisms (with appropriate reasons) for injecting code into processes as a DLL. Just look at all the categories in the [Sysinternals Autoruns](https://learn.microsoft.com/en-us/sysinternals/downloads/autoruns) tool. Many of those categories represents some way that code can register itself to be loaded into some or all processes. And that list certainly isn't exhaustive.

You can also get DLLs to be loaded by using the same name as one that's expected to be there, just earlier in the DLL search order. This is especially easy if the program attempts to load a DLL as a way of checking for some "optional" functionality. (This is a horrible practice, but believe me that it's everywhere. [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) can show you all the failed DLL load attempts). For those, all you need is a DLL in the current directory or on the PATH.

You might assume that DLL injection is more limited than patching, because you can't modify the behavior of the existing code. Combined with the power of [detours](https://github.com/microsoft/Detours) to change the behavior of an existing function (such as an OS function, say ```GetMessageW``` or <a aria-describedby="footnote-label" href="#tickcount">```GetTickCount64```</a>), and you now have the ability to drastically alter the behavior of a program!

Of course, DLL injection has its own set of downsides. It's the frequent source of application compatibility issues. If your DLL loads other DLLs, those DLLs can interfere with the target program. In one case, we got reports of a hardware OEM (of the LED-lit mouse/keyboard variety) that was triggering dbghelp.dll to be loaded into the WinDbg process before WinDbg could load it's *own* copy of dbghelp.dll, which caused all sorts of havoc. (Also due to the DLL search order, where modules in the loaded module list have precedence even over DLLs in the application directory when calling LoadLibrary!)

Some apps are also resistant to tampering via DLL injection. Chrome, for instance, has [some hardening](https://blog.chromium.org/2017/11/reducing-chrome-crashes-caused-by-third.html) that prevents many forms of DLL injection, as they saw significant stability issues caused by third party injection. Certain Windows processes can also [be protected](https://learn.microsoft.com/en-us/windows/win32/services/protecting-anti-malware-services-) when they have some security critical code .

# Dynamic code injection

Dynamic code injection is the trickiest form of code injection, and is generally reserved for nefarious purposes, although you will also see it getting used by diagnostic tools (such as early versions of Time Travel Debugging).

To do dynamic code injection without injecting a DLL, you allocate some memory in the target process using [AllocVirtualEx](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex), write some bytes representing the code you want to run using [WriteProcessMemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory), and then create a thread (or modify an existing thread) to get the process to run your code. You have to be careful what code you write to the remote process to make sure it can operate independently and not rely on code that *isn't* copied (or DLLs that are not loaded).

Let's take a look at an example. Note that this is a simplified example and won't work without careful compilation options. For one, we need to turn off the GS check (/GS-) because it will generate an RIP-relative memory access to data that won't be copied or allocated in the remote process. We'll also build in relase mode but disable optimizations (/Od). Given that, let's start with the payload.

```c++
using MbFunc = decltype(MessageBoxW);

DWORD WINAPI RemoteFunc(LPVOID lpThreadParameter)
{
    wchar_t sz[3];
    sz[0] = L'H';
    sz[1] = L'i';
    sz[2] = 0;
    MbFunc* mb = (MbFunc*)lpThreadParameter;
    mb(NULL, sz, sz, 0);
    return 0;
}
```

This function looks really weird, but all it does is call MessageBoxW with the string "Hi". It's written in a weird way to make sure it doesn't have any memory accesses outside of the function boundaries. It also takes the address of MessageBoxW as a parameter to avoid a memory access to the import table. We also copy the string in character by character to avoid a memory access to the globals section. This took a little trial and error looking at the code that got generated by the compiler, and it'd be pretty painful if you wanted to do anything more complex.

Now that we have a payload, we need to create a process and inject the code into the process. For that purpose, I'm just going to use winver.exe, as it is a simple win32 gui app with a single dialog.

```c++
    wchar_t szCmdLine[] = L"winver.exe";
    STARTUPINFOW si = {};
    PROCESS_INFORMATION pi = {};
    si.cb = sizeof(si);
    auto ret = CreateProcessW(NULL, szCmdLine, NULL, NULL, FALSE, 0, NULL, NULL, &si, &pi);
```

We can allocate some memory in the process using the handle returned:

```c++
    LPVOID ptr = VirtualAllocEx(pi.hProcess, NULL, 4096, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
```

Note that we use PAGE_EXECUTE_READWRITE. That's important! Without the EXECUTE bit, the code would immediately crash. We're allocating 4096 bytes, which will be more than enough memory for our function.

Now we calculate the size of the data to copy and write it using WriteProcessMemory:

```c++
    auto size = (char*)main - (char*)RemoteFunc;
    SIZE_T bytesWritten = 0;
    ret = WriteProcessMemory(pi.hProcess, ptr, RemoteFunc, size, &bytesWritten);
```

More weird code here. I've made an assumption that the compiler is going to lay out the ```RemoteFunc``` and ```main``` functions sequentially. That's a big assumption, but it works here. If you wanted to be more robust, it would be better to put the code in its own segment with the appropriate "pragamas", but I'm not going to worry about that for this example.

Finally, we need to actually execute the code. Since our function follows the same signature as LPTHREAD_START_FUNCTION, we can use CreateRemoteThread to start some code running our function. Also, remember that we need to pass in the address of MessageBoxW to the function, so we'll do that too.

```c++
    auto data = (char*)MessageBoxW;
    HANDLE hthread = CreateRemoteThread(pi.hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)ptr, data, 0, NULL);
```

You might be wondering how it's possible that we take the address of MessageBoxW in the local process, but call it in the remote process. This code relies on the fact that MessageBoxW is in user32.dll, and user32 will always be loaded at the same address for all processes on the system (along with ntdll and kernel32). This would not be safe with an arbitrary DLL, which can get relocated to different addresses in different processes.

Run this code and you have remote code injection on the winver.exe process, which you will see by a dialog box that says "Hi". And you can confirm that it's part of winver.exe process by closing the winver dialog, which will cause the message box to close as well!

# Conclusion

Hopefully this gave you some insights about how code injection can work on Windows. But please remember this is for educational purposes, and don't put any of the techniques I mentioned in real production code!

Have a question or suggestion? Let me know! You can find me on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim).

<footer>
  <h2 id="footnote-label">Footnotes</h2>
  <ol>
  <li id="tickcount">I once wrote a DLL that intercepted GetTickCount64 and a bunch of similar functions so that it could "stretch time" by some multiplier. It worked surprisingly well in a number of casual games with high score boards. Not that I would ever cheat of course...
  </li>
  </ol>
</footer>