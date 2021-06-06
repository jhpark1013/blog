---
layout: post
title: "Processes vs threads"
date: 2021-04-17T16:05:32-04:00
---
Processes vs threads, the main difference is that processes each have their own PID (i.e. process identification number: an identification number that is automatically assigned to a process when it is created) and addresses space. Threads share the PID and address space of a single process. Because threads write to the same memory, you have to have all kinds of race conditions which may require you to use mutexes.
