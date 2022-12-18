---
title: "What's a vtable? What's an IUnknown?"
date: 2022-12-18T08:30:09-07:00
draft: false
---

Understanding vtables (also called VMT/Virtual Method Tables or VFT/Virtual Function Tables) is important for understanding how many C++ features work on any OS. They are even more important to understand on Windows, where vtables are used for communication between modules using COM. There are a few types of errors that can happen when things go wrong with them, and some of these will be extremely difficult to diagnose unless you understand how vtables actually work.

I'll start this post with an overview of vtables and how they are related to COM objects. After that, I'll talk about some techniques you can use to examine a vtable in a debugger, and some of the things that can go wrong with code that uses vtables.

# What's a vtable?

The most basic form of a vtable is just a table of function pointers. When working with class hierarchies in C++ and similar languages, a vtable gives us a mechanism for "dynamic dispatch", or in other words, a way to decide at *runtime* which function should be called. If you have a class inheriting from another class, the derived class can provide its own implementation of any virtual functions.

{{< highlight cpp >}}
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
{{< / highlight >}}

We can use a ```ClassB``` as if it is a ```ClassA```, and the compiler will generate code to call the right version of ```ClassB``` at runtime. Consider the following C++ code:

{{< highlight cpp >}}
    ClassA* first = new ClassA();
    ClassA* second = new ClassB();
...
    auto firstNum = first->GetNumber();
    auto secondNum = second->GetNumber();
{{< / highlight >}}

Assuming the code is not so simple that the optimizer gets rid of any dynamic calls, the instructions the compiler generates will be looking at the virtual table to determine which function to call. With all optimizations disabled, MSVC produces something like this:

```
    auto firstNum = first->GetNumber();
00007FF767662607  mov         rax,qword ptr [first]  
00007FF76766260B  mov         rax,qword ptr [rax]  
00007FF76766260E  mov         rcx,qword ptr [first]  
00007FF767662612  call        qword ptr [rax]  
00007FF767662614  mov         dword ptr [firstNum],eax  
```

The first instruction gets a pointer sized value at the start of the ```ClassA``` object. This is the pointer to the virtual function table. The second line gets the first pointer size value of the virtual function table and stores it in ```rax```, which will be the address of the correct implementation of GetNumber. Then we get the address of the class again and store it in ```rcx```, which is where the calling convention specifies that the ```this``` pointer should be stored. Finally, we call the function we stored in ```rax```, which will be one of the implementations of GetNumber. The last assembly instruction takes the return value (which in this calling convention is in ```eax```) and stores it in the local variable called ```firstNum```

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

This gives us the pointer to the virtual function table (vftable) for ```ClassB```, which is part of the Win32TestApp. The table itself is part of the executable image, and not allocated on the heap or elsewhere in memory. A single vftable is shared by all instances of ```ClassB```. We can use the ```dps``` command once more to see the functions that are inside the table:

```
0:000> dps 00007ff7`daea6408 L1
00007ff7`daea6408  00007ff7`daea17c0 Win32TestApp!ClassB::GetNumber [Win32TestApp\Win32TestApp.cpp @ 23]
```

WinDbg shows that this resolves to the ```ClassB``` version of the ```GetNumber``` function, which is how the code ends up running the correct implementation. Note that if you try to follow along with this experiment you could see a different name here depending on your compiler options. Incremental linking is usually on by default, and will show a slightly different name here, but you should still see ```GetNumber``` in there.

# Vtables and "dynamic type deduction"

The structure of vtables allows for a neat trick that many debuggers take advantage of. Assuming we have ```ClassA``` and ```ClassB``` defined as they are in the last example, look at the following code snippet:

{{< highlight cpp >}}
void DoSomething(ClassA* ptr)
{
    std::cout << ptr->GetNumber() << std::endl;
}

int main()
{
    DoSomething(new ClassA());
    DoSomething(new ClassB());
}
{{< / highlight >}}

The symbolic information for the ```DoSomething``` function will say that the "static type" of ```ptr``` is ```ClassA```. A ```ClassB``` instance can be passed into the ```DoSomething``` function, and for these cases a debugger can look at the vtable for the object instance and determine whether the object is actually ```ClassA``` or ```ClassB```:

```
0:000> g
Breakpoint 0 hit
NativeApp!DoSomething+0x1f:
00007ff6`59351f9f 488b85e0000000  mov     rax,qword ptr [rbp+0E0h] ss:000000e8`2aaff680=000001f9fc383e10
0:000> dx ptr
ptr                 : 0x1f9fc383e10 [Type: ClassA *]
0:000> g
Breakpoint 0 hit
NativeApp!DoSomething+0x1f:
00007ff6`59351f9f 488b85e0000000  mov     rax,qword ptr [rbp+0E0h] ss:000000e8`2aaff680=000001f9fc386560
0:000> dx ptr
ptr                 : 0x1f9fc386560 [Type: ClassB * (derived from ClassA *)]
```

We refer to ```ClassB``` in this case as the "dynamic type" and ```ClassA``` as the "static type" of the object. Both VS and WinDbg can determine the dynamic type through the same technique, which is to look at the vtable for the object and determine whether is the functions of ```ClassA``` or ```ClassB```. In this case, it's pretty straightforward, but there are cases where optimizations can cause issues, such as when the compiler combines identical functions (sometimes called [COMDAT Folding](https://devblogs.microsoft.com/oldnewthing/20161024-00/?p=94575)). If the ```GetNumber``` functions are identical, they will get combined into a single function and the vtables for both ```ClassA``` and ```ClassB``` functions will look the same. So this is a heuristic that *can* go wrong sometimes. Generally though, there will be at least one function that is unique to the type and the logic from WinDbg and VS (which is essentially the same algorithm) is able to deduce the dynamic type.

# Vtables and stable ABIs

Everything I've described so far is specifically code generated by MSVC. Other compilers are free to implement the C++ language features in different ways. Testing this on [compiler explorer](https://godbolt.org) I saw that Clang had a similar layout, but GCC was different. So if every compiler (and version of a compiler) can have a different ABI for virtual function tables, how do you use across different modules that are not necessarily compiled with the same compiler? In the Windows world, the answer is COM.

Before you start screaming and run away, let me emphasize that while COM ultimately evolved into a hugely complicated system, the core idea is very simple and elegant. The core of COM starts with something called IUnknown<sup>1</sup>, a simple interface that represents a binary stable ABI for calling methods on objects across a set of languages and a way of managing the lifetime of objects used across multiple modules. IUnknown has three methods. Two of them are simply used for lifetime management through reference counting: AddRef and Release. The third method is what we're interested in, which is QueryInterface. This method takes a globally unique ID (GUID) as a parameter to identify a requested interface, and returns a pointer to the requested interface (if it is available) with a stable ABI.

This all works perfectly well, as long as you follow a few simple rules. The most important rule, however, is that an interface can NEVER CHANGE once it has been published with a GUID.<sup>2</sup>. Violating this rule is one source of crashes when dealing with COM interfaces. Crashes can show up as nonsensical stack trace where the source code is clearly calling a specific named function, but because the vtable has changed, a completely unrelated function gets called. 

# When vtables go wrong

The most frequent way you see vtables cause problems is when you get a mismatch between the expected layout of functions and the actual layout of functions. One way this can happen is with an incorrect cast. Take this code as an example:

{{< highlight cpp >}}
class ClassA
{
public:
    virtual int GetNumber()
    {
        return 4;
    }
};

class Class1
{
public:
    virtual float GetFloatingPointNumber()
    {
        return (float)4;
    }
};

_declspec(noinline) void DoSomething(ClassA* ptr)
{
    std::cout << ptr->GetNumber() << std::endl;
}

int main()
{
    DoSomething((ClassA*)(new Class1()));
}
{{< / highlight >}}

Running this program, the output I see is:

```
-1829428392
```

This happens because ```DoSomething``` calls the first method in the vtable, which it expects to be ```GetNumber```, but which is actually ```GetFloatingPointNumber```. The floating point value ```4.0``` gets interpreted as an integer, which is why you see the wrong value returned. If you look at this under a debugger, it can also be quite confusing because you'll see a call stack that doesn't really make sense:

```
0:000> kc
 # Call Site
00 NativeApp!Class1::GetFloatingPointNumber
01 NativeApp!DoSomething
02 NativeApp!main
```

It looks like ```DoSomething``` is calling into ```GetFloatingPointNumber```, but the source says that should never happen! If you don't understand how a vtable works, it might take you a very long time to understand what happened here. As a side note, using ```static_cast<ClassA*>(...)``` would have prevented this error, because the compiler won't let the invalid cast to compile.

# COM vtable mismatches

Vtable mismatches can happen in some subtle ways with COM, even if you think that you're following the rules. Suppose you have an interface called ```IWidget``` which gets returned from an ```IWidgetFactory```. After releasing your code, you decide that you need a new method on ```IWidget```, so you change the GUID for the interface (the Interface ID) and think that you're safe. But in reality, you've *also* changed the ```IWidgetFactory``` interface because it's not really the same ```IWidget``` being returned. Any callers that were built against the old version could get an ```IWidget``` that doesn't match with the expected vtable layout, causing the old code to call into completely unrelated functions. Hopefully it causes a crash, but in some cases the function signature is similar enough that the call still "works", but with the wrong behavior. This is an *incredibly* frustrating type of bug to track down, and there are a number of variants of this problem.

# Multiple inheritance

Most of this post has conveniently ignored multiple inheritance. If we have a class that derives from two different base classes (or base interfaces) with a different set of virtual functions, how do we get a vtable that looks like each of the base classes? The full answer would be a topic for an entirely separate blog post. But the short answer is that the compiler implements it through *multiple* vtables for the derived object. This also means that a cast of the derived object to a base class pointer is no longer a simple operation. I'll leave the rest for a future post.

# Conclusion

Understanding how virtual method tables work is important for diagnosing several classes of bugs in C++, as well as some classes of bugs that are specific to code on Windows that uses COM. Hopefully this post gives you enough information that you can recognize these issues when you see them and figure out how to fix them. Was this post interesting or useful? Let me know on [Twitter](https://twitter.com/timmisiak) or [Mastodon](https://dbg.social/@tim)!

## Footnotes

1. Yes, I realize there is also IDispatch and IInspectable, but those are each their own cans of worms.
2. This is why so many COM interfaces end up with a number. If you want to add a new function, you need a new interface, and extending the interface with a version number is the standard way of doing that. For instance, [IDebugClient5](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nn-dbgeng-idebugclient5)