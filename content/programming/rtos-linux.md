---
title: Real-Time Operating Systems and Linux
draft: false
tags:
  - operating-systems
---
I recently saw the [announcement of `PREEMPT_RT` entering the mainstream Linux kernel](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=baeb9a7d8b60b021d907127509c44507539c15e5) on RISC-V, arm64 and x86 after 20 years of development. `PREEMPT_RT` extensions allow the Linux kernel to be used in/as a real-time operating system (RTOS), changing the way task scheduling works. RTOS was something new for me, so I decided to do some research and put down by findings here!

### 1. What are operating-systems?
An operating-system is a software responsible for managing hardware resources of a computer and providing them efficiently to the applications that sit above it. The OS acts as a mediator between applications running on a computer and the *raw* hardware resources like memory, CPU, disk and I/O peripherals.

Operating systems are also responsible for resource management that ensures each application gets the requested resource in a definite time, avoiding race conditions and maximizing the utilization of that resource. The CPU is one such resource, which given performs operations by parsing instructions given to it.

### 2. Processes and scheduling

![[rtos-1.png|240]]

For an OS, each application is viewed as a set of processes. A process, which is a software construct, contains all necessary information needed to perform the task. It contains instructions (code) and the data on which the instructions work ([stack and the heap](https://stackoverflow.com/a/80113/13546426)), a program counter (a pointer indicating the current instruction being executed) and other meta-attributes. A process may require access to a resource, like a serial line to a peripheral or a socket or the CPU. 

As multiple applications (processes) need access to different resources, there must a mechanism that manages the interaction of the processes with the resources. A process scheduler is a component of the kernel that performs the exact task. The scheduler takes into account several parameters of the process, like, its *priority*, burst-time (estimated time to complete execution), arrival time (time when the process first requested the resource) etc. 

### 3. Time-sharing OSes
Operating systems like Ubuntu, Android and Windows which are widely used general-purpose OSes are time-sharing OSes. By 'time-sharing', we point to the fact that their schedulers try to distribute a given resource (like the CPU) to a maximum number of processes, where each process will get the resource for a fixed time-span. Each process gets the same *share of time* on the resource, which brings an illusion of simultaneous execution to the end user. 

For instance, we have 10 processes requesting a resource `X`, and we allow each process to hold the resource for `500 ms` sequentially until each process completes its use with `X`. This also enables multi-tasking, as 10 processes (applications maybe) do have access to the resource to carry out its operations (maybe not a continuous access, but still). 

![[rtos-3.png|500]]

For a CPU, a timer interrupt is generated at a fixed interval of time which instructs the CPU to perform a *context switch* and take up another process to execute. An interrupt is just a signal that instructs the CPU to pause its current execution and do something else (the *else* is defined by the [interrupt service routine](https://en.wikipedia.org/wiki/Interrupt_handler)).  

### 4. Downfalls of Time-Sharing and RTOS`
Using the same example, if the number of processes still remains 10, it is easy to predict how long the process will take to complete its execution. What if new processes keep coming requesting resource `X`, which happens in all real-world scenarios for a scheduler? In that case, the completion time is hard to guess!

What if the process needs to be completed in a specified time limit? For instance, if there's an request to `APPLY_BRAKES` in a embedded car computer, it needs to complete within a fraction of a second! The completion time needs to have an upper limit defined. There are many such applications where *instantaneous and bounded-time execution* is more important than resource utilization or multi-tasking. An OS which needs to handle such time-bound tasks need a special kind of scheduling mechanism which has a more deterministic policy to prioritize processes. 

A RTOS comes with these capabilities, making them a first-choice for time critical operations. A RTOS is event-driven and not *interrupt-driven*. It switches tasks when a high priority task arrives and puts an upper bound on the completion time of the process.

### 5. `PREEMPT_RT` 

![[rtos-2.png]]

`PREEMPT_RT` is a patch set (plugins) that enables a priority-based scheduler in the Linux kernel. This patch set was being developed independent from the main kernel, and it got included in the mainstream kernel on 20 Sept. 2024. 


### References
1. https://www.reddit.com/r/embedded/comments/tyccfj/differences_linux_and_real_time_linux/
2. https://ubuntu.com/blog/what-is-real-time-linux-i
3. https://ubuntu.com/blog/what-is-real-time-linux-ii
4. https://wiki.linuxfoundation.org/realtime/start
5. https://www.reddit.com/r/linuxquestions/comments/h8v3vg/preempt_rt_patch_for_linux/