---
title: volatile keyword in C/C++
tags:
  - programming
  - embedded-systems
---
Why do we often use the `volatile` keyword in C/C++ especially in embedded systems programming?

The `volatile` keyword in C/C++ informs the compiler that no optimizations should be made with the marked member. The value of member is not stored in registers and is always fetched from the memory, assuming that an 'invisible force' can change its value at any given moment. For embedded systems, as the executing program interacts closely with the peripheral and other hardware components, values in memory can be changed instantly. For example, providing a HIGH signal on a GPIO pin can change the value of a variable in the memory. The executing program will not be informed of this change and it will continue to execute with the former value of the variable.

To solve this problem, we mark the variable as 'volatile'. In the image above, the 'square' procedure contains a number of:

```
mov eax, DWORD PTR [rsp-4]
```

These calls are memory reads i.e. we read the variable 'num' directly from stack memory whenever it is required. The variable 'num' is marked as volatile, indicating that will not be stored in a register and will always be read from memory when needed. Hence, we observe many `DWORD PTR [rsp-4]` expressions in the `square` procedure.

In the `square2` procedure, the variable `num` is assumed to be in the register `rdi`. When a variable is stored in the register, an external hardware signal may not be able to modify it in place. Peripheral devices modifying memory is mostly the case for memory-mapped I/O.

![[Pasted image 20260809065748.png]]

Check [Compiler Explorer](https://godbolt.org/z/cfEaGsecx) here.