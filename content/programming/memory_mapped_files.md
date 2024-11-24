---
title: Exploring mmap and memory-mapped files in C
draft: false
tags:
  - programming
  - operating-systems
---
[`mmap`](https://man7.org/linux/man-pages/man2/mmap.2.html) is a Linux system call and a standard C library function which is used to create a mapping in the virtual address space (VAS) of the calling (current) process. 

* The mapping is **between the VAS and (optionally) a file/device** identified by a file-descriptor. 

* The **VAS of a process is set of addresses assigned by the operating-system (OS)** for use by the process. The addresses in the **VAS do not have a one-to-one mapping with addresses in the physical address space of the computer**; the latter being defined by the available physical random access memory (RAM) of the computer.

* The address space of the file/device is mapped directly byte-to-byte to a region belonging to the process's VAS. Such a file is referred as a **'memory-mapped' file**. The contents of a memory-mapped file can be accessed by dereferencing addresses present in the mapped region.

Here's a small C example which uses `mmap` to map the contents of a file `message.txt` in a read-only, private region in the process's VAS. The mapping can be observed using `pmap <pid>`. 

```c
// ex_mmap.c

#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <errno.h>
#include <unistd.h>

int main(int argc, char** argv) {
    // open a file descriptor
    int fd = open("message.txt", O_RDONLY);
    if (fd == -1) {
        fprintf(stderr, "error occurred in open(): %d", errno);
    }

    // create a mapping of the file contents in the virtual
    // address space of the process
    int page_size = getpagesize();
    void* vmem_mapping = mmap(
        NULL,           // let kernel choose a page-aligned address in the virtual memory
                        // space of the process
        page_size,      // map 'page_size' bytes from the file-descriptor
        PROT_READ,      // the mapping is read-only
        MAP_PRIVATE,    // the changes in this mapping are not visible to other processes
        fd,             // file-descriptor to map
        0               // offset in file-descriptor from which to start mapping
    );

    // read chars starting from address `vmem_mapping`
    // and print them to stdout until a '0' char is read
    size_t i = 0;
    char file_char;
    while ((file_char = *(char*)(vmem_mapping + i)) != '0') {
        putchar(file_char);
        i += 1;
    }

    // wait until a 'q' is entered on stdin
    // this helps us look into the process's memory
    // using pmap <pid>
    while (getchar() != 'q');

    // unregister the mapping we created
    munmap(vmem_mapping, page_size);

    return 0;   
}
```

Compiling and running the executable,

```bash
> touch message.txt
> echo "Hello world from the virtual address space!" >> message.txt
> gcc ex_mmap.c -o ex_mmap
> ./ex_map
Hello world from the virtual address space!
```

Using `top`, we get the process-identifier (PID) of the process and pass to `pmap <pid>`,

```
> ps aux | grep -i ex_mmap
shubham   1025  0.0  0.0   6336  1940 pts/5    D+   11:07   0:00 grep -i ex_mmap
shubham  32531  0.0  0.0   2468   920 pts/4    S+   11:03   0:00 ./ex_mmap
```

```
> pmap 32531
> 32531:   ./ex_mmap
00005581bd62e000      4K r---- ex_mmap
00005581bd62f000      4K r-x-- ex_mmap
00005581bd630000      4K r---- ex_mmap
00005581bd631000      4K r---- ex_mmap
00005581bd632000      4K rw--- ex_mmap
00005581e444a000    132K rw---   [ anon ]
00007fa13a20c000     12K rw---   [ anon ]
00007fa13a20f000    152K r---- libc.so.6
00007fa13a235000   1364K r-x-- libc.so.6
00007fa13a38a000    332K r---- libc.so.6
00007fa13a3dd000     16K r---- libc.so.6
00007fa13a3e1000      8K rw--- libc.so.6
00007fa13a3e3000     52K rw---   [ anon ]
00007fa13a3f6000      4K r---- message.txt
00007fa13a3f7000      8K rw---   [ anon ]
00007fa13a3f9000      4K r---- ld-linux-x86-64.so.2
00007fa13a3fa000    148K r-x-- ld-linux-x86-64.so.2
00007fa13a41f000     40K r---- ld-linux-x86-64.so.2
00007fa13a429000      8K r---- ld-linux-x86-64.so.2
00007fa13a42b000      8K rw--- ld-linux-x86-64.so.2
00007ffeb810f000    136K rw---   [ stack ]
00007ffeb81be000     16K r----   [ anon ]
00007ffeb81c2000      8K r-x--   [ anon ]
 total             2468K
```

The mapping named `message.txt` is visible alongside `libc` and `ld-linux-x86-64` which are shared libraries whose functions are referred by our program. 

> [!INFO]
> Why is the `ld-linux-x86-64` (GNU Linker)'s code mapped to the process's VAS? 
> 
> `ld-linux-x86-64` is a dynamic linker which loads shared library dependencies at runtime from persistent storage to the main memory of the computer. It not only links the target executable to the shared libraries but also places machine code functions at specific address points in memory that the target executable knows about at link time. When an executable wishes to interact with the dynamic linker, it simply executes the machine-specific call or jump instruction to one of those well-known address points.

### Why do we need to `mmap` files?

![[mmap_files_01.png]]

- Reading/writing files with `fopen` and `fwrite` (which internally use `read` and `write` system calls) is buffered at multiple stages between the current program and the storage device. There exist buffers in the kernel-space and user-space code resulting in multiple copies of the file. With a memory-mapped file, a direct access to the page-cache (file-cache) of the operating system is made, significantly reducing the sys-call overhead.

- The pages belonging to the mapped region are also loaded lazily i.e. only when they're required by the process ([demand paging](https://en.wikipedia.org/wiki/Demand_paging))

- The page-cache does not contain all requested pages, thus resulting in [page-faults](https://en.wikipedia.org/wiki/Page_fault). For larger files where frequent accesses are made randomly (not sequentially), the number of page-faults can increase, hurting the performance of the operation.





