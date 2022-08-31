---
title: "What's the Debugger Data Model? (And Why?)"
date: 2022-08-30T23:26:09-07:00
draft: false
---

If you follow [me on Twitter](https://twitter.com/timmisiak), you have probably heard me talk about the "Debugger Data Model". But unless you've spent a bunch of time [reading our documentation](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/using-linq-with-the-debugger-objects) or you've read articles such as [this one by Yarden Shafir about the Data Model](https://medium.com/@yardenshafir2/windbg-the-fun-way-part-1-2e4978791f9b), you might have no idea what I'm talking about.

Many of the ways that WinDbg is evolving to be more powerful are through the Debugger Data Model, and understanding it will help you use WinDbg more effectively. So in this post, I'm going to write a bit about what the data model is, and why we made it.

# WinDbg's greatest strength and biggest weakness: Extensions

If you use WinDbg a lot, then there's a pretty good chance that you've used some debugger extensions. Extensions are a big part of why WinDbg is so useful. There are hundreds of extension commands that ship with the debugger, and thousands more available. They can do everything from displaying internal kernel structures to inspecting the state of the .NET managed heap. Despite how useful they are, debugger extensions as they have existed for the past few decades have some problems.

**The first problem** is that there are simply too many debugger extensions to find the right ones to use. You can learn about useful extensions from articles, books, coworkers, and documentation, but the debugger is rarely going to tell you the right extension to use for the problem that you're looking at. Hyperlinks between extension commands (what we call ["DML"](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-markup-language-commands)) go a long way towards making it possible to find related extensions. But even with the hyperlinks, you still have to know where to start. In short, traditional debugger extensions are **not discoverable**.

**The second problem** is that the old style of debugger extensions **don't compose because extensions have unstructured output**. In other words, you can't easily use the information in one extension and chain that with another extension. People have written text processing and chaining extensions, but none of these are going to work smoothly with every extension. And since the output is unstructured, it's not something you'll be able to easily use if you want to script the debugger or create a conditional breakpoint. 

**The third problem** is that debugger extensions are **incredibly difficult to write**. Extensions are generally written using the [DbgEng COM-style API](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nn-dbgeng-idebugclient) or a wrapper on top of these APIs such as [EngExtCpp](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/engextcpp-extension-libraries). Some tasks are painful, and others are downright impossible.

Even with all of these problems, extensions have become a huge part of WinDbg's success. So how do we address these problems?

# The Debugger Data Model

Our answer to this was the Debugger Data Model. We wanted to make extensions **easy to discover**, **easy to compose**, and **easy to write**. To do that required a shift in how we think about debugger extensions. The biggest part of that shift is moving from a text model to a structured data model. From this, we have a path to solve the three biggest problems with text only extensions.

# Discoverability

The best way to make debugger extensions discoverable is to simply make them available when they are needed without having to even know about them in the first place. Previously, to debug any of the STL data structures you would either need to manually interpret the contents of structures (which could be painful for things like hash tables) or you would need to know about extensions such as ```!stl```. With the debugger data model, the extension for interpreting the STL structures gets used *automatically* whenever you look at an STL structure. 

Take this code for example:

{{< highlight cpp >}}
int main()
{
    std::vector<std::wstring> testVec;

    testVec.push_back(L"This is the first string");
    testVec.push_back(L"Here's string #2");
    testVec.push_back(L"And our final string!");
}
{{< / highlight >}}

If we debug this and break in right after the last ```push_back``` call, we see something like this when we look at the local variables using the [dx command](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/dx--display-visualizer-variables-) which evaluates objects using the data model.

```
0:000> x
000000b1`f39bf468 testVec = { size=3 }
0:000> dx testVec
testVec                 : { size=3 }
    [<Raw View>]     [Type: std::vector<...>]
...
    [0]              : "This is the first string" 
    [1]              : "Here's string #2"
    [2]              : "And our final string!"
```

Compare that to what you would see without the debugger data model. We can use the ```??``` command, which by default does not use the debugger data model for evaluation and visualization.

```
0:000> ?? testVec
class std::vector<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,std::allocator<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >  > >
   +0x000 _Mypair          : std::_Compressed_pair<std::allocator<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > >,std::_Vector_val<std::_Simple_types<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > > >,1>
```

The difference is that the debugger extension for visualizing STL types is used automatically when you use the data model through ```dx```. You don't need to know that the extension exists. What's more, there are actually two different debugger extensions being used here. One for visualizing the ```std::vector``` and another for visualizing the ```std::wstring```. These two extensions work together in a natural way. It doesn't matter that they are both visualizers for STL types. Any extensions such as these will compose in the data model because they describe structured data, instead of simply text output. Which brings us to the next aspect of the debugger data model.

# Structured data and composability

Making extensions in the debugger data model compose together naturally means that the extensions deal with structured data, not text. More than that, it gives us the ability to have debugger extensions that can be used in [conditional breakpoints](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-a-conditional-breakpoint). It gives us the ability to write [scripts that query data from extensions](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/javascript-debugger-scripting). And it also gives us the ability to [filter, query, and visualize data in a standard way](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/dx--display-visualizer-variables-).

Consider code that does something like this.

{{< highlight cpp >}}
std::vector<int> testVec;

void doSomething()
{
    if (rand() % 3 == 1)
    {
        testVec.push_back(rand());
    }
}

int main()
{
    for (int j = 0; j < 100; j++)
    {
        doSomething();
    }
}
{{< / highlight >}}

If we want to set a breakpoint on the doSomething() function so that we break into the debugger only when the testVec structure has 10 items, we can set it with a condition.

```
0:000> bp /w "testVec.Count() == 10" doSomething
0:000> g
Breakpoint 0 hit
WindowsConsoleApp!doSomething:
00007ff6`e6476c50 4055            push    rbp
0:000> dx testVec.Count()
testVec.Count()  : 0xa
```

Here we've created a conditional breakpoint that takes advantage of the extension that understands STL containers in a way that we would never have been able to do with a legacy style extension. Beyond just conditional breakpoints, you can write your own scripts that interact with other extensions and build on them. It's easy enough that it sometimes makes sense to write one-off scripts in cases where you would never dream of writing an old style debugger extension. Which brings us to the last point I want to make.

# Ease of use

A major design goal for the Debugger Data Model was to have a much easier model for writing all sorts of debugger extensions. For that, there are three main ways of writing extensions. The easiest is [NatVis](https://docs.microsoft.com/en-us/visualstudio/debugger/create-custom-views-of-native-objects?view=vs-2022), which can be used to create visualizations for native objects. NatVis provides a concise way of expressing the logical structure of an native object so that it can be cleanly visualized. For instance, the std::vector visualizer looks something like this in NatVis.

{{< highlight xml >}}
<Type Name="std::vector&lt;*&gt;">
  <DisplayString>{{size = {_Mylast - _Myfirst}}}</DisplayString>
  <Expand>
    <Item Name="[size]">_Mylast - _Myfirst</Item>
    <Item Name="[capacity]">(_Myend - _Myfirst)</Item>
    <ArrayItems>
      <Size>_Mylast - _Myfirst</Size>
      <ValuePointer>_Myfirst</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>
{{< / highlight >}}

NatVis has a number of advantages, including the ability to be embedded in a PDB, and support in both WinDbg and Visual Studio. You can edit [NatVis files directly in WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/windbg-scripting-preview) in a very streamlined workflow.

NatVis will always be my first choice when writing visualizations. If NatVis isn't expressive enough, or if you want to write a debugger extension that does more than just interpret a native structure, you can write an extension in JavaScript. For instance, if you are debugging some code with a vector of structures like this:

{{< highlight cpp >}}
struct foo
{
    int iVal;
    bool bVal;
};

std::vector<foo> testVec;

int main()
{
    testVec.push_back(foo{ 5, false });
    testVec.push_back(foo{ 100, true });
    testVec.push_back(foo{ 20, false });
    testVec.push_back(foo{ 30, true });
    testVec.push_back(foo{ 1, true });
}
{{< / highlight >}}

You can write a script that access this data as simply as this:

{{< highlight js >}}
function findSumOfTrueItems(vec)
{
    var sum = 0;
    for (var item of vec)
    {
        if (item.bVal)
        {
            sum += item.iVal;
        }
    }
    return sum;
}
{{< / highlight >}}

And you can load this script and execute it in the debugger:

```
0:000> dx @$scriptContents.findSumOfTrueItems(testVec)
@$scriptContents.findSumOfTrueItems(testVec) : 0x83
```

In this example, you can see that the script works with other extensions. The NatVis visualizer lets the debugger bring the ```std::vector``` object into the JavaScript environment seamlessly. The individual members are native objects, and also get projected into JavaScript, making it very easy to access data of the target that is being debugged. It's simple and concise. And it doesn't stop with just native objects. You can also write extensions that add information to concepts such as processes, threads, and stack frames, adding data to queries like ```dx @$curprocess``` or ```dx @$curstack.Frames```.

Going into more depth than this would make this post way too long, but if you want to learn more, you can take a look at the [documentation](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/javascript-debugger-scripting) as well as our [samples](https://github.com/microsoft/WinDbg-Samples/tree/master/DataModelHelloWorld), which include both JavaScript and C++ data model extensions.

# Conclusion

Even without going into much depth, this post has gotten long. There's a lot more I could say here, and I'm sure I'll write more about specific aspects of the data model. We still have lots of plans for how to make the debugger data model easier to use and more powerful, and I don't think we've even scratched the surface of what it can become.

What else do you want to know about the data model? Have any scripts or queries that you think are cool? Let me know what you think on [Twitter](https://twitter.com/timmisiak)!

