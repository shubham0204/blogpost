---
title: Memory Tagging - A Memory Safety Technique
tags:
  - operating-systems
---
## Introduction
Memory bugs such as heap-buffer-overflow and heap-use-after-free are often exploited by malicious actors to perform actions that may leak sensitive user data or disrupt the functioning of the software. These bugs are common in languages with no automatic memory management like C and C++. 

Memory tagging is a technique included in the Arm ISA v8.5 specification to improve memory safety by associating a 'tag' with allocated memory blocks. 

- The tag is applied to the memory block and to the pointer variable holding the address of the memory block. 

- Pointers use 'address tags' that are stored in the top 4 bits of every pointer used in the process.

- For memory blocks, each granule i.e. a 16-byte region is annotated with a 4-bit tag referred as a 'memory tag'. Logically, every 16 bytes of memory now contain an extra 4 bits of metadata in addition to 128 bits of data.

## Working
The following describes how memory tags are modified during memory operations:

- Allocation: The operating system assigns a random tag to the allocated memory block and to the pointer holding the address of the block.

- Access: For all subsequent read and write operations, the tag stored within pointer/address in matched against the tag assigned to the memory block. If the tags do not match, the CPU signals the operating-system and operation is blocked immediately (via a `SIGSEV`) or reported asynchronously.

- Deallocation: The operating-system changes the tag of the allocated memory block and marks it as 'free' for further allocations.

## Preventing Memory Bugs - Examples

### Heap Buffer Overflow

![[mt_heap_buffer_overflow.png]]

`ptr` is assigned the tag `A` and the corresponding memory block is assigned the same tag. When trying to access a memory location outside the allocated block i.e. `ptr[32]` the tags of the `ptr` `A` and that of the memory location, `E`, mismatch, leading to a `SIGSEV`.

### Heap Use After Free

![[mt_heap_use_after_free.png]]

`ptr` and the corresponding memory block are assigned the tag `D`. When deallocating the memory block i.e. calling `free(ptr)`, the tag of memory block is changed from `D` to `4`. Now, if the memory block is access with `ptr` for ex. with `ptr[16]`, the tags mismatch and the access is refused by the operating-system.
## Comparison with Address Sanitizer
Address Sanitizer, commonly referred as `Asan`, is another technique that protects a program against illegal memory accesses at runtime. When `Asan` is attached/embedded into the program, each memory allocation is surrounded by guard-rails that are blocks of memory managed by `Asan`.

`Asan` generally takes up more memory than memory-tagging as it uses additional memory-blocks as red-zones (guard-rails).

Here's an excellent blog to understand how Address Sanitizer works: https://suelan.github.io/2020/08/18/20200817-address-sanitizer/#Heap-buffer-overflow

## References
- https://security.apple.com/blog/memory-integrity-enforcement/
- https://www.usenix.org/system/files/login/articles/login_summer19_03_serebryany.pdf
- https://developer.arm.com/documentation/108035/0100/Introduction-to-the-Memory-Tagging-Extension?lang=en
