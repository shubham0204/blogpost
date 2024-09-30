---
title: Exploring Smart Pointers in C++
draft: false
tags:
  - programming
  - cpp
---
Smart pointers are types of template classes in C++ that provide a secure way to allocate and de-allocate memory. By *''smart'*, we mean that these pointers are lifetime-aware and *know* when to de-allocate themselves, without explicitly calling `delete` or `delete[]`. In larger codebases, keeping track of all heap-allocated objects is difficult and failing to call `delete` at the right instances can lead to memory leaks or undefined behavior. 

Smart pointer classes encapsulate the raw pointer holding the address of the object in memory along with some added mechanisms that help them automatically deallocate the memory through the raw pointer. They also help in managing the ownership of the object which dictates how the object will be modified or de-allocated. In this short blog, we'll explore four smart pointer classes, `auto_ptr`, `unique_ptr`, `shared_ptr` and `weak_ptr`.  

## RAII (Resource Acquisition Is Initialization)
[RAII](https://en.cppreference.com/w/cpp/language/raii) is a principle which originated in C++ and then used in other languages like Rust and Ada. It suggests that the memory acquired by an entity (an object) must be tied to the lifetime of the object. Meaning, when the object is created, its memory must be initialized and when the lifetime of the object ends, the memory must be returned back (or the object must be de-allocated). **RAII is also called 'Scope Bound Resource Management**, which largely justifies the principle. 

Instead of allowing the programmer to manually allocate/de-allocate memory for an object, the responsibility is transferred to the object which uses its lifetime as a means to make decisions regarding the memory. In the upcoming sections, we will observe smart-pointers being used to follow the RAII principle in C++.

Before we start, here's the code for the dummy `HTTPConnection` class that will be used to demonstrate object creation/deletion using smart-pointers in the later sections.

```cpp
#include <memory>
#include <iostream>
#include <string>

class HTTPConnection {
public:
	std::string host;
    int port;

    HTTPConnection(const std::string host, int port) : host(host), port(port) {
        std::cout << "Connection (" << host << "," << port << ") created" << std::endl;
    }

    ~HTTPConnection() {
        std::cout << "Connection (" << host << "," << port << ") destroyed" << std::endl;
    }
}
```
## Using `auto_ptr`
An object enclosed in an [`auto_ptr`](https://en.cppreference.com/w/cpp/memory/auto_ptr) gets deleted when the `auto_ptr` instance (smart pointer instance) goes out of scope. Here's a snippet of code that demonstrates one of the use-cases of an `auto_ptr`,
```cpp
void download() {
    // A dumb pointer
    HTTPConnection* connection1 = new HTTPConnection("www.google.com", 8000);

    // A smart pointer (auto_ptr)
    std::auto_ptr<HTTPConnection> connection2(new HTTPConnection("www.github.com", 8000));

    // send/receive data from the connection
    // forgot to deallocate connection1 and connection2 here ...
}

int main(int argc, char* argv[]) {
    download();
    return 0;
}
```
Both, `connection1` and `connection2`, point to data allocated on the heap. To deallocate `connection1`, we need to *manually* call the `delete connection1` whereas `connection2`, being a smart pointer, detects the end of its scope (the scope of both pointers is the function `download` where they reside) and deletes the enclosed object *automatically*.
```text
Connection (www.google.com,8000) created
Connection (www.github.com,8081) created
Connection (www.github.com,8081) destroyed
```

Next, consider a function that accepts some parameters and returns the pointer to a heap-allocated object initialized with the given parameters.
```cpp
HTTPConnection* alloc_return_ptr(
    const std::string& host,
    int port
) {
    HTTPConnection* connection = new HTTPConnection(host, port);
    return connection;
}

int main(int argc, char* argv[]) {
    HTTPConnection* connection = alloc_return_ptr("www.github.com", 8000);
    std::cout << connection << '\n';

    HTTPConnection* connection_copy = connection;
    std::cout << connection << '\n';
    std::cout << connection_copy << '\n';

    // A double free error?
    delete connection;
    // ...
    delete connection_copy;
    
    return 0;
}
```
The *dumb* pointer `connection` returned from `alloc_return_ptr` and copied multiple times, where each copy (similar to `connection_copy`) can be used to modify the object in memory and even deallocate it! Hence, copying a *dumb* pointer results in the creation of several owners of the same object that may lead to unusual consequences in a large codebase.
```
Connection (www.github.com,8000) created
0x564e75d352f0
0x564e75d352f0
0x564e75d352f0
Connection (www.github.com,8000) destroyed
Segmentation fault
```
Notice, when deallocating `connection_copy` leads to a `SEGFAULT` as the underlying object is  already deleted by `delete connection`. This is an example of the [double free memory leak](https://en.wikipedia.org/wiki/Memory_safety#:~:text=Double%20free%20%E2%80%93%20repeated%20calls%20to%20free%20may%20prematurely%20free%20a%20new%20object%20at%20the%20same%20address.%20If%20the%20exact%20address%20has%20not%20been%20reused%2C%20other%20corruption%20may%20occur%2C%20especially%20in%20allocators%20that%20use%20free%20lists.).

`auto_ptr` comes with *destructive copy semantics*, meaning, when a `auto_ptr` is copied, the original `auto_ptr` instance gets invalidated (set to `nullptr`) and the copy gets the valid address of the object. Thus, the ownership of the object is transferred completely to the new copy, thus preserving the *one-object, one-owner* principle.
```cpp
std::auto_ptr<HTTPConnection> alloc_return_auto_ptr(
    const std::string& host,
    int port
) {
    std::auto_ptr<HTTPConnection> connection(new HTTPConnection(host, port));
    return connection;
}

int main(int argc, char* argv[]) {
    std::auto_ptr<HTTPConnection> connection = alloc_return_auto_ptr("www.google.com", 8000);
    std::cout << connection.get() << '\n';

    std::auto_ptr<HTTPConnection> connection_copy = connection;
    std::cout << connection.get() << '\n';
    std::cout << connection_copy.get() << '\n';

    return 0;
}
```
Output:
```
Connection (www.google.com,8000) created
0x55d4dc4aaeb0
0
0x55d4dc4aaeb0
Connection (www.google.com,8000) destroyed
```
Notice, printing `connection.get()` before creating a copy and after creating a copy yields different values. `connection` is set to `0` after `connection_copy` is created. 
> [!CAUTION]
> `std::auto_ptr` was deprecated in C++11 and removed in C++17, in the favor of `std::unique_ptr` and `std::shared_ptr`.

## Using `unique_ptr`
A [`unique_ptr`](https://en.cppreference.com/w/cpp/memory/unique_ptr) can be used as a replacement for `auto_ptr` as it has the same goal for maintaining one-owner per heap-allocated object. We can understand `unique_ptr` by understanding its differences from the now-deprecated `auto_ptr`.  The following code does not compile,
```cpp
std::unique_ptr<HTTPConnection> connection(new HTTPConnection("www.github.com", 8081));
std::unique_ptr<HTTPConnection> connection1 = connection;
```
A `unique_ptr` cannot be copied like an `auto_ptr`, but *moved* to another `unique_ptr` thus transferring ownership. For the above code to compile, we can use `std::move`,
```cpp
std::unique_ptr<HTTPConnection> connection(new HTTPConnection("www.github.com", 8081));
std::unique_ptr<HTTPConnection> connection1 = std::move(connection); // Move-assignable ALLOWED
std::unique_ptr<HTTPConnection> connection2(connection.release()); // Move-constructible ALLOWED
// std::unique_ptr<HTTPConnection> connection3 = connection; // Copy-assignable NOT ALLOWED
// std::unique_ptr<HTTPConnection> connection4(connection1); // Copy-constructible NOT ALLOWED
return 0;
```
> [!INFO] Difference between *moving* and *copying* objects
> A [good SO answer](https://stackoverflow.com/a/64379301/13546426) suggests *"Imagine an object containing a member pointer to some data that is elsewhere in memory. For example, a `std::string` pointing at dynamically allocated character data. Or a `std::vector` pointing at a dynamically allocated array. Or a `std::unique_ptr` pointing at another object. A _copy_ constructor must leave the source object intact, so it must allocate its own copy of the object's data for itself. Both objects now refer to different copies of the same data in different areas of memory (for purposes of this beginning discussion, lets not think about reference-counted data, like with `std::shared_ptr`). A _move_ constructor, on the other hand, can simply "move" the data by taking ownership of the pointer that refers to the data, leaving the data itself where it resides. The new object now points at the original data, and the source object is modified to no longer point at the data. The data itself is left untouched. That is what makes [_move semantics_](https://stackoverflow.com/questions/3106110/what-is-move-semantics) more efficient than _copy/value sematics_."*

`std::move` and `unique_ptr::release` make the transfer of ownership more explicit and the use of move-semantics over copy/value-semantics improves efficiency. To create a `unique_ptr`, the `std::make_unique` function can also be used, which creates the `unique_ptr` and the underlying object's instances in one single memory allocation.
```cpp
std::unique_ptr<HTTPConnection> connection = std::make_unique<HTTPConnection("www.github.com", 8081);
```
## Using `shared_ptr`
Instead of one heap-allocated object having just one owner, a [`shared_ptr`]() allows multiple owners. The enclosed object is deallocated once the **last** `shared_ptr` goes out-of-scope or when the last pointer is transferred to another pointer. 
```cpp
std::shared_ptr<HTTPConnection> connection(new HTTPConnection("www.github.com", 8081));
std::shared_ptr<HTTPConnection> connection1 = connection;
std::shared_ptr<HTTPConnection> connection2 = connection;

// Use one of the references to modify the object
connection2 -> host = "www.google.com";

// Use other references to access the object, and
// check for modifications
std::cout << connection -> host << '\n';
std::cout << connection1 -> host << '\n';
```
For the object `("www.github.com", 8081)`, we have three references `connection`, `connection1` and `connection2` that can access and modify the underlying object. The object is automatically deallocated when these three references go out of scope,
```text
Connection (www.github.com,8081) created
www.google.com
www.google.com
Connection (www.google.com,8081) destroyed
```

Just like `make_unique`, there exists the `std::make_shared` function to create a `shared_ptr` + object instance in one memory allocation. 
### `weak_ptr`
Quoting from [cppreference.com](https://en.cppreference.com/w/cpp/memory/weak_ptr), `std::weak_ptr` is a smart pointer that holds a non-owning ("weak") reference to an object that is managed by `std::shared_ptr`. It can be used to access the object only if it exists and has not been deallocated already (see [dangling pointers](https://en.wikipedia.org/wiki/Dangling_pointer)).

By making use of the `expired()` method, one can check if the managed object is valid. The `lock()` method creates a `shared_ptr` to the object if it is valid. If any part of the code is holding a `shared_ptr` for an object, the object cannot be deleted until the code relieves its `shared_ptr`. With a `weak_ptr`, the code can still access the object if required with actually owning it. 

Here's an excellent answer on [StackOverflow](https://stackoverflow.com/a/12030687/13546426) highlighting a use-case of `weak_ptr`,

>A good example would be a cache.
>
>For recently accessed objects, you want to keep them in memory, so you hold a strong pointer to them. Periodically, you scan the cache and decide which objects have not been accessed recently. You don't need to keep those in memory, so you get rid of the strong pointer.
>
>But what if that object is in use and some other code holds a strong pointer to it? If the cache gets rid of its only pointer to the object, it can never find it again. So the cache keeps a weak pointer to objects that it needs to find if they happen to stay in memory.
>
>This is exactly what a weak pointer does -- it allows you to locate an object if it's still around, but doesn't keep it around if nothing else needs it.
## Closing Thoughts
Smart pointers felt fascinating from the day I saw them being used in a code snippet. After learning Rust, I could understand how concepts like ownership and RAII work. With smart-pointers, I feel these concepts can be applied in C++ too. Not a C++ expert, but this blog will serve as my 'notes' while I was learning smart-pointers in C++. 

Keep learning, and have a good day ahead!

