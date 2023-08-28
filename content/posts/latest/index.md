---
title: "Building Your Own C++ library for Linux: A Guide For Beginners"
description: "Transforming your .cpp files to #include libraries"
tags: [ "c++" ]
category: [ "c++" ]
showToc: true
date: 2023-07-04T21:23:49+05:30
draft: false
---



## Understanding The Structure of C++ Programs

C++ packages consist of two types of files, source and header files. Generally, header files contain declaration for classes/functions, whereas source files contain the corresponding implementation. So, why is the declaration separated from the implementation? 

Imagine that going out on a Sunday dinner. Once you enter the restaurant, the waiter provides you a menu, which contains a list of all dishes along with their ingredients, spice-level categorized into multiple cuisines. The dishes aren't made ready in front of you, but only a overview of the dish is presented with the menu card. Just as the menu contains only 'descriptions of the dishes', header files contain only the 'declaration' of the function, which mentions,

* name
* arguments and their data-type
* return-type

These parameters describe the function completely, just as the menu-card describes the dish at a restuarant. Whereas, the implementation of the functions is analogous to the food that will be served.

### Creating a header file for our `linkedlist` library

A header file (with a `.h` or `.hpp` extension) conventionally holds class or function declarations that could be used by other C/C++ programs.


The `linkedlist.h` file will contain the following code:

```cpp
// linkedlist.h
#include <vector>

// A Node object will hold the data 
// and a pointer to the next Node in the list
class Node {

    int val ; 
    Node* next = nullptr ; 

    public:

    friend class LinkedList ; 

} ;

// The core LinkedList class that all other programs will use
class LinkedList {

    Node* ROOT = nullptr ; 

    public:

    // Class method declarations for LinkedList
    void insert( int element ) ; 
    void remove( int element ) ; 
    void search( int element ) ; 
    std::vector<int> to_vector() ; 

} ; 
```

Next, we implement the above declared methods in a source file.

### Creating a source file for our `linkedlist` library

The source file contains the implementation 


```cpp
// (source) linkedlist.cpp
#include "linkedlist.h"

void LinkedList::insert( int element ) {
    if( ROOT == nullptr ) {
        ROOT = new( Node ) ; 
        ROOT -> val = element ; 
    }
    else {
        Node* currentNode = ROOT ; 
        while( currentNode -> next != nullptr ) {
            currentNode = currentNode -> next ; 
        }
        Node* newNode = new( Node ) ; 
        newNode -> val = element ; 
        currentNode -> next = newNode ; 
    }
}
```

```cpp
bool LinkedList::search( int element ) {
    Node* currentNode = ROOT ; 
    while( currentNode != nullptr ) {
        if( currentNode -> val == element ) {
            return true ; 
        }
        currentNode = currentNode -> next ; 
    }
    return false ; 
}
```

```cpp
std::vector<int> LinkedList::to_vector() {
    std::vector<int> output ;
    Node* currentNode = ROOT ; 
    while( currentNode != nullptr ) {
        output.push_back( currentNode -> val ) ; 
        currentNode = currentNode -> next ; 
    }
    return output ; 
}

LinkedList::~LinkedList() {
    Node* currentNode = ROOT ; 
    while( currentNode != nullptr ) {
        Node* nextNode = currentNode -> next ; 
        delete currentNode ; 
        currentNode = nextNode ; 
    }
}
```
## Compilation and Linking

Considering C/C++, compilation is the process of transforming each translation unit (i.e. a source or a header file) to an object-code file. The object-codes of multiple translation units have to be linked together to form an executable or a shared library. So, C++ programs are built in two phases:

* Compilation: Transforming source code to object code (`.obj`)
* Linking: Transforming object code to executables or libraries

### Compilation

In our case, the contents of the header file will be included as-is in the source file, so we're left with only a translation unit. We use `g++` to produce object files for the `linkedlist.cpp` and `linkedlist.h`, by specifying the `-c` flag, which instructs the compiler to terminate the compilation process once the object files are created. If this flag is not specified, the compiler will create an executable directly. 

```commandline
g++ -c linkedlist.cpp -o linkedlist.o
```

The compiler could find `linkedlist.h` automatically because of the `#include` directive in `linkedlist.cpp`. After the execution of the statement, an object code `linkedlist.o` is generated. 

### Linking

Next, we use the [GNU Linker]() `ld` to link `linkedlist.o` and produce a shared object (`.so`) which is a dynamic library. Dynamic libraries (or shared libraries) can be shared by multiple programs and each executable, which uses a dynamic library, loads it at runtime. This means the library's code isn't a part of the executable and the shared library needs to be present on the system where the executable is shipped.


```commandline
ld -shared linkedlist.o -o liblinkedlist.so
```



