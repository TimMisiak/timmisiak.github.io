---
title: "What's a VTable? What's an IUnknown?"
date: 2022-11-26T08:30:09-07:00
draft: true
---

Understanding VTables (also called VMT/Virtual Method Tables or VFT/Virtual Function Tables) is important for understanding how many C++ features work on any OS, but even more important on Windows, where this is a common mechanism used for communication between modules. There are also a class of errors that can happen when things go wrong, and some of these will be extremely difficult to understand what has gone wrong unless you understand some of the fundamentals for how these things work.

I'll start this post with a very brief overview of VTables and how they are related to COM objects. After that, I'll talk about some techniques you can use to examine a vtable in a debugger, and some of the things that can go wrong with code that uses vtables.

# What's a VTable?

The most basic form of a vtable is just a table of function pointers. When working with class heirarchies in C++ and similar languages, a vtable gives us a mechanism for "dynamic dispatch", or in other words, a way to decide at *runtime* which function should be called. If you have a class inheriting from another class, the derived class can provide its own implementation of any virtual functions.

```
class ClassA
{
public:
    virtual int GetNumber()
    {
        return 4;
    }
};

class ClassB : public ClassA
{
public:
    virtual int GetNumber()
    {
        return 5;
    }
};
```

We can use a "ClassB" as if it is a ClassA, and the compiler will generate code to call the right version of ClassB at runtime. Consider the following C++ code:

```
    ClassA* first = new ClassA();
    ClassA* second = new ClassB();
...
    auto firstNum = first->GetNumber();
    auto secondNum = second->GetNumber();
```

Assuming the code is not so simple that the optimizer gets rid of any dynamic calls, the instructions the compiler generates will be looking at the virtual table to determine which function to call. With all optimizations disabled, MSVC produces something like this:

```
    auto firstNum = first->GetNumber();
00007FF767662607  mov         rax,qword ptr [first]  
00007FF76766260B  mov         rax,qword ptr [rax]  
00007FF76766260E  mov         rcx,qword ptr [first]  
00007FF767662612  call        qword ptr [rax]  
00007FF767662614  mov         dword ptr [firstNum],eax  
```

The first instruction gets a pointer sized value at the start of the ClassA object. This is the pointer to the virtual function table. The second line gets the first pointer size value of the virtual function table and stores it in ```rax```, which will be the address of the correct implementation of GetNumber. Then we get the address of the class again and store it in ```rcx```, which is where the calling convention specifies that the ```this``` pointer should be stored. Finally, we call the function we stored in ```rax```, which will be one of the implementations of GetNumber. The last assembly instruction takes the return value (which in this calling convention is in ```eax```) and stores it in the local variable called ```firstNum```

To see this in a slightly different way, we can examine the object in WinDbg. If we launch this executable and examine one of these objects with the ```dt``` command (dump type), we see:

```
0:000> dt ClassA
Win32TestApp!ClassA
   +0x000 __VFN_table : Ptr64 
```

WinDbg doesn't hide the fact that there's an element here called __VFN_table, which is the virtual function table. If we take a look at a specific instance, we can follow the pointer into the virtual function table, and then examine the table itself.

```
0:000> dx second
second                 : 0x26b57d1bbd0 [Type: ClassB * (derived from ClassA *)]
```

We find the address of the object, and then dump the first element using ```dps``` (dump pointer sized elements and resolve symbols)

```
0:000> dps 0x26b57d1bbd0 L1
0000026b`57d1bbd0  00007ff7`daea6408 Win32TestApp!ClassB::`vftable'
```

This gives us the pointer to the virtual function table (vftable) for ClassB, which is part of the Win32TestApp. The table itself is part of the executable image, and not allocated on the heap or elsewhere in memory. A single vftable is shared by all instances of ClassB. We can use the ```dps``` command once more to see the functions that are inside the table:

```
0:000> dps 00007ff7`daea6408 L1
00007ff7`daea6408  00007ff7`daea17c0 Win32TestApp!ClassB::GetNumber [Win32TestApp\Win32TestApp.cpp @ 23]
```

WinDbg shows that this resolves to the ClassB version of the GetNumber function, which is how the code ends up running the correct implementation. Note that if you try to follow along with this experiment you could see a different name here depending on your compiler options. Incremental linking is usually on by default, and will show a slightly different name here, but you should still see "GetNumber" in there.

# VTables and stable ABIs

Everything I've described so far is specifically code generated by MSVC. Other compilers are free to implement the C++ language features in different ways. Testing this on [compiler explorer](https://godbolt.org) I saw that Clang had a similar layout, but GCC was different. So if every compiler (and version of a compiler) can have a different ABI for virtual function tables, how do you use across different modules that are not necessarily compiled with the same compiler? In the windows world, the answer is COM.

Before you start screaming and run away, let me emphasize that while COM ultimately evolved into a hugely complicated system, the core idea is very simple and elegant. The core of COM starts with something called IUnknown<sup>1</sup>, a simple interface that represents ideas of a binary stable ABI for calling methods on objects across a set of languages and a way of managing the lifetime of objects used across multiple modules. IUnknown has three methods. Two of them are simply used for lifetime management through reference counting: AddRef and Release. The third method is what we're interested in, which is QueryInterface. This method takes a globally unique ID (GUID) as a parameter to identify a requested interface, and returns a pointer to the requested interface (if it is available) with a stable ABI.

This all works perfectly well, as long as you follow a few simple rules. The most important rule, however, is that an interface can NEVER CHANGE once it has been published with a GUID.<sup>2</sup>. Violating this rule is one source of crashes when dealing with COM interfaces. Crashes can show up as nonsensical stack trace where the source code is clearly calling a specific named function, but because the vtable has changed, a completely unrelated function gets called. 

# Conclusion

The idea of recognizing a sequences of bytes and finding the meaning behind them has always felt very satisfying to me. Have you ever found this useful? Do you have any tricks of your own? Let me know on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim)!

## Footnotes

1. Yes, I realize there is also IDispatch and IInspectable, but I'm ignoring those for the sake of brevity.
2. This is why so many COM interfaces end up with a number. If you want to add a new function, you need a new interface, and extending the interface with a version number is the standard way of doing that. For instance, [IDebugClient5](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nn-dbgeng-idebugclient5)