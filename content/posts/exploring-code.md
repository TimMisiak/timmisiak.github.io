---
title: "Exploring Code"
date: 2022-08-15T15:16:09-07:00
draft: true
---

For many people, a debugger is the tool of last resort. Maybe they try printf debugging, logs, or other tracing first. And only if those fail do they break out a debugger. For many systems, that's a reasonable approach. But for me, a debugger is more than a tool of last resort. It's the tool I turn to first when I'm first starting to understand a codebase. An IDE can help you understand the structure of the code, but often falls short when trying to understanding the flow of data. An IDE will tell you all the places a function is referenced, but not what values it gets called with during normal operation. So in this post, I'm going to give you a peek into how I use a debugger to understand a codebase.

# The codebase: Notepad++

I'm going to start with the codebase of Notepad++. I've been reading some of the code there recently to see if I can use it as examples for debugging different types of bugs. But as soon as I opened the code, I saw something interesting:

```
cmdLineParams._easterEggName = getEasterEggNameFromParam(params, cmdLineParams._quoteType);
```

So Notepad++ has an easter egg! But what is it, and how does it work? 