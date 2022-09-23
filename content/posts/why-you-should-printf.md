---
title: "Why you should do printf debugging"
date: 2022-09-18T13:20:24-07:00
draft: true
---

If you know who I am, you probably think that my blog post title is clickbait. Well it is, kind of. But the truth is, you should do printf debugging! Sometimes. Usually not. But sometimes, you should!

Let me explain. When most of us first started programming, we had one tool at our disposal. It was "printf debugging". Sometimes just littering the code with ```printf("here")```, ```printf("here2")```, and my favorite, ```printf("why won't this code work?!?!?!")```. Then we spent hours staring at the console output trying to figure out why some code did or didn't execute, or try to figure out what's going on in our code based on the values that were being printed out.

Later, some of us discovered debugging, and realized that for many cases it was easier to set a breakpoint than to recompile the code every time we had a new theory. There are also some who never found a use for a debugger. Linus Torvalds [famously said](https://lkml.org/lkml/2000/9/6/65):

```
I don't like debuggers. Never have, probably never will. I use gdb all the
time, but I tend to use it not as a debugger, but as a disassembler on
steroids that you can program.
```

Linus clearly is clearly a very intelligent and talented engineer, and his perspective is worth understanding. If you read his full <s>rant</s> email, his main point seems to be around putting up a barrier to entry to force people to think more carefully about the code they are writing. I can understand why he thinks it's valuable to have barriers to entry on contributing to the Linux kernel, but I don't feel that logic works as well for cases where we want to make it **easier** for people to contribute features and fix bugs. But I think that the larger point is that debuggers are not always the right tool for the job. And that's what this post is about. Finding the right tool for the job is important, because using the right tools will make you more effective at troubleshooting problems.

# Tradeoffs

For most problems that you will diagnose, there will be multiple paths to get to the right answer. There are overlaps between the types of information that each tool gives you access to, but the method for collecting the data will affect how efficiently you can analyze the problem. Both printf debugging and profilers can give you access to similar sets of information, but the types of problems that are appropriate to debug with each are very different because they make different tradeoffs about the data that is collected. The tradeoffs I think about for diagnostic tools are between lightweight vs. heavyweight, time vs. space, and predefined vs. ad hoc.

## Lightweight vs. Heavyweight

I frequently try to start with the most lightweight tool that I think will give me the answers. When supporting customers, that usually means I start with logs. Probably 75% of problems that I look at for WinDbg customers can be solved with logs alone. Since our internal error reports include logs by default, it means there is a very low cost to collecting the data, and I can respond to customer issues extremely quickly. The downside of course is that there is less information and the remaining problems I need to ask the customer for more information, generally a [Time Travel Debugging](https://aka.ms/ttd) (TTD) trace or more context so I can reproduce the issue. This takes a lot more time, so of course it makes sense to solve as many problems as possible using lightweight techniques such as logging.

To take this a step further, there are even heavier weight techniques such as live kernel debugging sessions. Whenever possible, I'd rather use a live usermode session or even a kernel dump if I think it will give me enough information to diagnose the problem because a live kernel debugging session is expensive, both in terms of my time and my customer's time. What it means to be a "heavyweight" tool is usually measured in time, either for a developer or a customer, but sometimes it is also measured in computational cost or storage requirements. A TTD trace, for instance, may be too computationally intensive for some scenarios. A full kernel dump of a large server might result in a file that is difficult to store or transmit.

## Time vs. space

Some debugging techniques are designed to analyze small amounts of data over longer periods of time, such as logging or tracing. Other debugging techniques are designed to analyze large amounts of data at a single point in time, such as a full crash dump or screenshot. Yet other techniques are designed to analyze a large amount of data over longer periods of time, such as live debugging sessions or a Time Travel Debugging trace.

Generally, techniques that prioritize "time" are more useful when debugging performance issues or otherwise incorrect behavior. Techniques that prioritize "space" are usually better for crashes and hangs. The most difficult problems may require both, such as a crash which is dependent on a longer chain of events. This sometimes requires combining two techniques, such as a crash dump combined with log files, or a single technique like a TTD trace.

## Predefined vs. ad hoc

When writing an application or other system, it's often clear where some of the potential failure points are, or where there is important information in order to understand the context of what is happening. In those cases, logging is the perfect solution. We aren't psychic, though, and so it's never the case that we always know what information will be needed at the time we are writing the code. So we also have "ad hoc" data collection that will be specific to something that you are troubleshooting. Sometimes that means a "printf" that you add to track the state of a vriable. Other times it's using a dynamic tracing tool such as [DTrace](https://en.wikipedia.org/wiki/DTrace).

Both approaches are important. Without good logging, nearly every issue will require in-depth analysis. Ad hoc instrumentation/tracing lets you dig into the specifics for the issue that you're looking at.

# So... when should you use printf debugging?

In the context of these tradeoffs, when should you be using printf debugging? You should be using printf debugging when you need a light weight, ad hoc method of examining a narrow set of data or code flow over a longer period of time. If you are using printf debugging in any other context, there's a good chance that there's a better tool for the job. Grow the set of tools that you're using so that you can use the right tool for the job.

