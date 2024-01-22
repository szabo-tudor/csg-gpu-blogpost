---
layout: default
title: "The Pitfalls of Porting Code to the GPU"
---


## What This Article is About
When writing code for a platform for the first time, it's easy to assume that the platform's standards won't change too much from what you are used to.
The logic usually stays the same, regardless of what programming language, compiler/interpreter, and OS you use.

This is where GPUs differ from this "standard", in that when you write a shader, it might not be executed the same on a different machine, or on the same machine with a different OS or different version of the same OS.


## Why This Topic
I chose this topic because, as a relative beginner to programming for GPUs, there are a few mistakes I've made that took me a long time to notice. Those mistakes caused problems way later down the line, in different places, making those mistakes a lot harder to find and fix.

These mistakes are not always obvious, and sometimes they're not even mistakes on some platforms. Nvidia GPUs have different standards for things like memory alignment, priorities, profiling etc, than AMD GPUs for example.
