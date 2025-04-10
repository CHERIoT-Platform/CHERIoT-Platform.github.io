---
layout: post
title:  "CHERIoT and Microvium"
date:   2025-04-10
categories: javascript vm
author: David Chisnall
---

We've included a [port](https://github.com/CHERIoT-Platform/cheriot-rtos/tree/main/sdk/include/microvium) of the [Microvium](https://microvium.com) embedded JavaScript runtime.
We originally did this port even before we open sourced the CHERIoT project
We haven't talked about it much and that's something of an omission, since it is quite a nice case study in supporting a managed language on a CHERI platform

# It Just Worked™

The first thing to note is that the initial 'port' didn't require any code changes.
We were able to take the Microvium codebase, compile it, and run it in a compartment, unmodified.

This is a nice result because language runtimes are traditionally some of the most difficult things to port to CHERI platforms.
We don't get to take credit for that, it came from the fact that Microvium was written as portable C code.
Language runtimes written for larger systems often do a lot of things that are tailored for specific operating systems or architectures.

# Pointers are 15 bits!

In some ways, Microvium is very similar to a classic Smalltalk-80 [Blue Book](http://stephane.ducasse.free.fr/FreeBooks/BlueBook/Bluebook.pdf) implementation.
Values are 16 bits and are either numbers or pointers, differentiated by a tag bit.
This means that a pointer in Microvium is a 15-bit value.
This can address up to 64 KiB of RAM (pointers refer to 16-bit words, not bytes).

We didn't want to increase the memory size for the CHERI port.
Going from 16-bit values to 64-bit ones would have quadrupled the memory consumption.
Fortunately, there was no need to.
Using 16-bit values on a platform with 64-bit capabilities to a 32-bit address space worked fine.

Microvium has two modes for memory management.
The first assumes that you are targeting a *really* tiny device and running bare metal.
Here, you reserve a chunk of memory for the JavaScript heap and pointers are just added to that base address.

Alternatively, for hosted environments, it allocates memory from some system-provided allocator.
Pointers are now offsets within a linear address space composed from walking the list of chunks.
This seems slow, but remember that Smalltalk-80 ran a complete interactive GUI with a similar amount of memory to a modern CHERIoT system but a processor around a thousandth the speed of a CHERIoT Ibex, so we can afford to waste a few cycles.

# Making Microvium a library

Microvium is designed for embedded targets and, in particular, for being able to instantiate multiple JavaScript VMs on a device.
Each one needs a couple of hundred bytes of stack and global context, a similar amount of bytecode memory, and usually a KiB or so of heap (more for complex programs).
On most systems, the code for the interpreter is shared between them and we wanted to be able to use Microvium in the same way.

This required building Microvium as a *shared library*.
Doing this at all required one code change in Microvium: adding a `MVM_EXPORT` macro to the functions exposed in the header file so that we could mark them with the `__cheriot_libcall` macro.
This let us build the VM as a library.
It wasn't quite enough to make it *work* as a library.
The VM also needed to be able to allocate memory.

On CHERIoT, the C `malloc` function is a wrapper around `heap_allocate`, which takes an explicit capability that authorises allocating against a quota.
We needed a mechanism to pass this quota from the calling compartment down to the malloc functions.
Microvium added a hook that allowed callers to pass a context parameter into the VM-creation function.
This context value was then passed to the allocate and deallocate functions each time Microvium called them.
Both of these changes [landed in the same PR upstream](https://github.com/coder-mike/microvium/pull/52).

With this, we could build a single copy of the Microvium VM and share the code between multiple compartments.

# Bounding pointers passed to C

A few of the Microvium APIs expose pointers to C code.
These originally spanned an entire Microvium heap slab and were read-write.
We [added two hooks to allow ports to provide bounds and make the regions immutable](https://github.com/coder-mike/microvium/pull/80).

With these two changes, if you pass a string from JavaScript to C (for example), the C code receives a read-only capability with the correct bounds.
This gives you greater confidence that bugs in your FFI layer can't break type safety in the JavaScript code.

# Temporal safety for C and JavaScript

Microvium uses a copying garbage collector.
Their implementation has one very nice property that makes it integrate with the CHERIoT temporal safety mechanism trivially: It does not move objects within a chunk.

Microvium allocates memory from the system in chunks (ports can configure the size).
The garbage collector finds live objects and copies them to *new* chunks and frees the old ones.

This means that a pointer from C to JavaScript is always in one of three states:

 - It points to a live JavaScript object.
 - It points to a garbage (but not collected yet) JavaScript object.
 - It points to a deallocated chunk.

In the first two cases, the pointer continues to point to a valid object and will work.
In the third state, the chunk is gone and so the pointer's tag bit will be cleared (by the CHERIoT load filter and / or revoker), so attempts to access it from C/C++ will trap.

# Lessons for other managed-languages on CHERI platforms

Microvium happened to be exactly the right shape to make a CHERIoT port easy.
It's optimised for low memory consumption at the expense of performance (the right trade for embedded devices, where CPU performance has increased at a rate far greater than memory size) and these choices avoided a lot of tricks that don't directly translate to CHERI platforms.
The compressed-pointer representation meant that Microvium already had a notion of internal and host pointers as distinct things (something it shares with a lot of managed-language VMs), which is a convenient place to apply CHERI bounds and restrict permissions.

Importantly, if you trust the implementation of your type-safe language, you don't need to make every pointer a capability *internally*.
We kept 16-bit (15-bit + tag) pointers within the JavaScript interpreter, but we extended them to full capabilities at the boundary.
This lets the VM provide type safety internally and the hardware provide it for FFI code.
This is often the right approach for managed languages on CHERI, unless the VM is so complex that you want additional defence in depth from memory-safety bugs.

CHERI systems can provide temporal safety for C and GC implementations that avoid memory reuse can be simply layered on top.
On larger CHERI systems, GCs may be able to use the same underlying mechanisms as the C allocators but they'll have the same issues: you can't reuse memory immediately, until you're sure that C code hasn't reused it.
This means that things like semispace compacting collectors (which eagerly reuse memory) are a problem, but mark-and-compact approaches that copy objects to new chunks are fine.

This kind of integration was why I started working on CHERI 13 years ago: to be able to write code in safe languages, reuse the enormous amount of code available in C/C++, and not lose the safety properties of the safe language.
It always makes me happy to see evidence that we've achieved this goal.
