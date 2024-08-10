---
layout: post
title: "OS - virtual memory"
subtitle: "why do we need virtual memory"
category: Operating System
---

## why we need virtual memory?

#### 1. Abstraction of physical memory

Consider a situation where we handle physical memory directly. Our code might vary depending on the amount of physical memory available. This means that our code has a dependency on the physical memory size.

We can't be sure the amount of memory available on the computer running our program. It's ridiculous that our code needs to vary based on the computer it's running on.

By adopting virtual memory, we can eliminate the dependency on physical memory. Instead of using physical memory addresses directly, we can allocate the virtual memory that processes require. We just need to manage the mapping between virtual memory addresses and physical memory addresses (`MMU` - Memory Management Unit).

#### 2. Facilitation of memory management

Consider a situation where a process crashes due to incorrect memory access. We need to free the memory sector (or segment or page) that the process was using. If memory is managed with an MMU, this is simple: you just erase the mapping information from the MMU. Similarly, virtual memory makes allocation and release of memory easier for the operating system.
