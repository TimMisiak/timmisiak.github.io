---
title: "Debugging Linux core files"
date: 2022-08-25T09:57:31-07:00
draft: true
---

There's a new trick that WinDbg has... blah blah blah...


```
tim@tmisiak-z2:~$ gcc -g test.c
tim@tmisiak-z2:~$ ./a.out
Hello world!
Floating point exception
tim@tmisiak-z2:~$ ls
a.out  test.c
tim@tmisiak-z2:~$ ulimit -S -c unlimited
tim@tmisiak-z2:~$ ./a.out
Hello world!
Floating point exception (core dumped)
tim@tmisiak-z2:~$ ls
a.out  core  test.c
```

navigate to \\wsl$\Ubuntu\home\tim

open core file from windbg preview.

A \\wsl$\Ubuntu\home\tim to sympath and srcpath
