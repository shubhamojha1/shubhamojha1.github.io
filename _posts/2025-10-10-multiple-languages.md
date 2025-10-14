---
layout: post
title: How some projects use Multiple Programming Languages
date: 2025-06-02 00:00:00
description: a lot more goes on with system calls than one would imagine
tags: formatting links
categories: sample-posts
---

System calls are the only way user programs request services from the kernel (e.g. file I/o, process creation, networking). Every time a program calls read(), open(), fork(), etc., it traps into the kernel. This boundary crossing incurs overhead. A context switch, register marshalling, nad possible pipeline flushes. 

The Linux kernel provides tricks to make common system calls cheaper. Frequent calls like gettimeofday(), clock_gettime() are implemented via vDSO (virtual dynamic shared object), a small library that the kernel maps into every process. 

the kernel is what ensures that you dont accidently delete 

https://www.youtube.com/watch?v=XJC5WB2Bwrc&t=1s