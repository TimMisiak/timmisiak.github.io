---
title: "Why you should do printf debugging"
date: 2022-09-24T12:20:24-07:00
draft: false
---

If you know who I am, you might think that this post title is clickbait. Maybe it is, a little. But the truth is, you *should* do printf debugging! Sometimes. Often not. But sometimes, you should!

Let me explain. When most of us first started programming, we had one tool at our disposal. It was "printf debugging". Sometimes just littering the code with ```printf("here")```, ```printf("here2")```, and my favorite, ```printf("why won't this code work?!?!?!")```. Then we spent hours staring at the console output trying to figure out why some code did or didn't execute, or try to figure out what's going on in our code based on the values that were being printed out.

Later, some of us discovered debugging and realized that it was easier to set a breakpoint than to recompile the code every time we had a new theory. There are also some who never found a use for a debugger. Twenty years ago Linus Torvalds [famously said](https://lkml.org/lkml/2000/9/6/65):

```
I don't like debuggers. Never have, probably never will. I use gdb all the time, but I tend to use it not as
a debugger, but as a disassembler on steroids that you can program.
```

Linus clearly is clearly a very intelligent and talented engineer, and his perspective is worth understanding. His main point is that it's good to put up a barrier to entry for contributing to the Linux kernel. He believes that it's useful to force people to think more carefully about the code they are writing. I can understand why he thinks it's valuable to have barriers to entry on contributing to the Linux kernel, but I don't feel that logic works as well for cases where we want to make it **easier** for people to contribute features and fix bugs. But I think that the larger point is that debuggers are not always the right tool for the job. And that's what this post is about. Finding the right tool for the job is important, because using the right tools will make you more effective at troubleshooting problems.

# Tradeoffs

For most problems that you will diagnose, there will be multiple paths to get to the right answer. There are overlaps between the types of information that each tool and technique gives you access to, but the method you use for collecting information will affect how efficiently you can analyze the problem. Both printf debugging and profilers can give you access to similar data, but the types of problems that are appropriate to debug with each are very different. This is because they make different tradeoffs about the data that is collected. The tradeoffs I think about for diagnostic techniques are lightweight vs. heavyweight, time vs. space, and predefined vs. ad hoc.

## Lightweight vs. heavyweight

I always try to start with the most lightweight tool that I think will give me the answers. When supporting customers, that usually means I start with logs. The majority of problems that I look at for WinDbg customers can be solved with logs alone. Since our internal error reports include logs by default, it means there is a very low cost to collecting the data, and I can respond to customer issues extremely quickly. The downside of course is that there is sometimes not enough information to debug the issue. For the problems that can't be debugged using logs, I need to ask the customer for more information, which is generally a [Time Travel Debugging](https://aka.ms/ttd) (TTD) trace, a crash dump, or simply more details on how reproduce the issue. Digging in with more heavyweight tools will always take more time, so it makes sense to solve as many problems as possible using lightweight techniques such as logging.

To take this a step further, there are even heavier weight techniques such as live kernel debugging sessions. Usually I'll avoid using kernel debugging whenever some alternative is viable, because a live kernel debugging session is expensive both in terms of my time and my customer's time. What it means to be a "heavyweight" tool is usually measured in time, either for a developer or a customer, but sometimes it is also measured in computational cost or storage requirements. A TTD trace, for instance, may be too computationally intensive for some scenarios. A full kernel dump of a large server might result in a file that is difficult to store or transmit.

## Time vs. space

Some debugging techniques are "time-oriented" and are designed to analyze smaller amounts of data over longer periods of time, such as logging or tracing. Other debugging techniques are "space-oriented" and are designed to analyze larger amounts of data at a single point in time, such as a full crash dump or screenshot. Yet other techniques are designed to analyze a large amount of data over longer periods of time, such as live debugging sessions or a Time Travel Debugging trace.

Generally, techniques that prioritize "time" are more useful when debugging performance issues or otherwise incorrect behavior. When you have no specific anchor to a point in time (such as a breakpoint, crash, or exception), it's effective to start with a technique that collects data over some period so that you can narrow in on a point of time to use with another tool. Techniques that prioritize "space" are usually for crashes and hangs where you have an anchor point in time to start from. The most difficult problems may require both, such as a crash which is dependent on a longer chain of events where you would use a time-oriented tool to get an anchor point in time and switch to a space-oriented tool to understand the state of the program at that point in time. For instance, adding printf logging to compare the difference between a "good" and "bad" execution of some scenario may help you narrow in on a point in time to set a breakpoint or capture a crash dump to examine in a debugger. Alternatively, for these types of problems you may use a single technique like Time Travel Debugging which captures both the time and space dimensions, but makes other tradeoffs by being heavier weight than logging or crash dumps.

## Predefined vs. ad hoc

When writing an application or other system, it's often clear where some of the potential failure points are, or where there is important information in order to understand the context of what is happening. In those cases, logging is the perfect solution. We aren't psychic, though, and so it's never the case that we always know what information will be needed at the time we are writing the code. So we also have "ad hoc" data collection that will be specific to something that you are troubleshooting. Sometimes that means a "printf" that you add to track the state of a variable. Other times it's using a dynamic tracing tool such as [DTrace](https://en.wikipedia.org/wiki/DTrace).

Both approaches are important. Without good logging, nearly every issue will require in-depth analysis. Ad hoc instrumentation/tracing lets you dig into the specifics for the issue that you're looking at.

# Why are there so many zealots?

One thing that I've noticed is that people are often adamant about the superiority of their particular diagnostic-technique-of-choice. Proponents of printf debugging often scoff at people who use debuggers, and vice versa. There are a few things that drive that.

In some parts of Microsoft, especially in teams that came out of the Windows organization, there is a very heavy emphasis on debugging. That drives a cycle where debugging effectiveness is improved (such as through debugging extensions and experience), and other techniques such as logging are not improved as much. This widens the gap and continues to make debugging even better while other techniques are less useful.

In the Linux world, I think you see the opposite. Linus Torvalds and others working on Linux are more skeptical of debuggers and more supportive of things like logging and tracing. And those techniques got more powerful for Linux users over time in a virtuous cycle. Many people from this sort of background see how powerful logging and tracing tools are for their platform, and think that debugging is an inferior technique.

Sometimes it's not so much the platform but the domain that you work in that influences your perspective. Someone working on a large scale system where failures are probabilistic is going to see things very differently than someone who works on a JIT that dynamically generates machine code. It's easy to fall into the trap of thinking that your domain is representative of every scenario, and I think this is what creates "zealots" who think their view is the only "correct" way to think. Of course, this isn't a helpful state of mind. Sometimes the right answer is to use a debugger, and sometimes the answer is printf (or something else altogether!)

# So... when should you use printf debugging?

Given all this, when should you be using printf debugging? I think it's helpful to think in terms of the tradeoffs. You should be using printf debugging when you need a light weight, ad hoc method of examining a narrow set of data or code flow over a longer period of time. If you are using printf debugging in other contexts, maybe there is better tool for the job. Grow the set of tools that you're using and you'll be more effective at solving problems.

