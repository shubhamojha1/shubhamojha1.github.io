---
layout: page
title: Bifrost - Hash Join Engine
description: Implementation of Hash Join Engine with multiple hash functions and benchmarking
img: assets/img/projects/bifrost.png
importance: 1
category: work
related_publications: true
---

{% include figure.liquid path="assets/img/projects/bifrost.png" title="Bifrost - Hash Join Engine" class="img-fluid rounded z-depth-1" %}
## Bifrost
this project started out as just an implementation of hash joins. implementing various kinds of joins, collision resolution strategies, benchmarking and tracking memory consumption. however, all this lead me to explore databases into even more detail. so i decided to build my own database. lets see how this turns out. you can find the code [here](https://github.com/shubhamojha1/bifrost).

i will keep updating this page. and add references as i find and use them.

There is a cold war between the OS people and Database people. Seeing that OS people control all the hardware and have their own rules for executing system calls, the database people cannot trust the workings of the os. for example, fsync and mmap.