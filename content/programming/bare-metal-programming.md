---
title: A Glimpse Into Bare Metal Programming
tags:
  - programming
  - operating-systems
---
### Intuition

Executables compiled with languages like C, C++ or Rust may look independent at the first glance, as we do not require any [virtual machine](https://stackoverflow.com/questions/4640809/what-is-a-vm-and-why-do-dynamic-languages-need-one) to run them, which is needed for programs written in Python or Java. Operating systems use a loader, a system software which is responsible for bringing instructions and data into the main memory for execution. The loader also brings essential routines and functions which are required for execution, such as the C or C++ standard library.

Executables do request [object code](https://en.wikipedia.org/wiki/Object_code) of functions that they're going to use from these standard libraries, and hence their presence in the operating system is essential. GNU releases of C/C++ standard libraries are predominant in most Linux distros, and hence we rarely notice them as dependencies of an executable. 

> What if the operating system is absent, and these standard library routines are unavailable?

### Bare-Metal Applications

Applications which run in a such a 'bare metal' environment when interaction is made directly with the hardware are usually complex, but form a crucial part of [embedded systems](https://en.wikipedia.org/wiki/Embedded_system). Due to the absence of a operating system, the [bootloader](https://en.wikipedia.org/wiki/Bootloader) of the computer needs to be instructed from where the execution of the program must be started.

Moreover, [standard input (`stdin`) or output streams (`stdout`)](https://en.wikipedia.org/wiki/Standard_streams) are absent and hardware signals need to be generated in order to validate the program's state while executing in a bare-metal environment. Building such applications may require additional steps, as all standard library routines required by the program need to be supplied with the executable, with no external links.

| ![[bare-metal-programming-01.jpg]]  |
| ------------------------ |
| <center>*Dynamic links for a simple 'hello world' compiled executable*</center> |

Observe the libraries `libc.so` (C-standard library) and `ld-linux-x86-64.so` (GNU Linker) which are loaded at runtime.

Bare-metal programming is an interesting field, nested deeply within embedded systems development. Interaction with the hardware directly provides more control to the developers and sustains more resources due to the absence of the operating system (which indeed also needs resources to execute).