---
layout: post
title:  "Avoiding stack overflows on CHERIoT"
date:   2024-05-01
categories: rtos stack programming
author: "David Chisnall"
---

If you put code in a compartment and do nothing to protect the interfaces, there are a lot of ways that a caller can make that compartment crash.
In some cases, this doesn't matter.
The compartment exists to provide confidentiality and integrity, but not necessarily availability.
The TLS compartment in the network stack is a good example of this.
It provides strong flow isolation and so making it crash will impact only the TLS session of the caller.
The caller can also simply call the close function on a TLS session and so we don't worry about their ability to break a TLS session, only about their ability to send plain text over the network or extract keying material from the TLS session.

In other contexts, availability is far more important.
In the core of the RTOS, the scheduler and allocator both have availability requirements.
If you can crash the allocator while it holds the lock over the heap, you can prevent any future memory allocation.
If you can crash the scheduler in the middle of updating run queues, you may be able to prevent certain threads ever running.

In most platforms, it's easy to make a function crash by moving the stack pointer to near the end just before calling it.
The function will run off the end of the stack and hit a guard page on MMU / MPU systems or the end of the stack capability on CHERI systems.
This is even more crucial on embedded systems, where stacks tend to be small.
Large desktop or server systems often have stacks of 1-4 MiBs, which are large enough for most programs to treat as infinite.
Embedded systems may have stacks that are under 1 KiB.
Even without the security implications from distrust between compartments, having systems fail because they ran out of stack space is far from ideal.

These problems are relevant on CHERIoT.
The caller can constrain the amount of space available on the stack before a cross-compartment call.
If the callee requires more stack space than the caller provides then this can cause problems.

From the start, the CHERIoT ABI has provided some mitigation for this.
Every cross-compartment entry point is described by an entry in an export table.
One of the fields in an export-table entry indicates the amount of stack space that a call requires.
If the stack has less space than is available then the call will fail without invoking the callee at all.

This leads to an obvious problem: How do you set this value to something sensible?
By default, the compiler sets it to the stack space required for the function that implements the entry point.
This means that any function that doesn't call any other functions is fine.
It also means that, if you put stack checks in the function *before* calling any other functions, then you can guarantee that they will work correctly.

This still leaves a lot of work to determine how much stack space you actually need.
The compiler or other tooling could build a static control-flow graph for the current compilation unit, and possibly even the current compartment, but what happens if you call library functions?

It turns out that we already had the building block for a good solution.
CHERIoT guarantees that you can't accidentally leak data left on the stack through a cross-compartment call.
This is done by zeroing the portion of the stack that is going to be shared before and after a cross-compartment call.
When we did this initially, we quickly realised that we were spending a lot of time zeroing memory that was already full of zeroes.
This got worse the larger stacks were.
If you had a 4 KiB stack and a function that used 128 bytes, you may still end up zeroing almost 4 KiB *twice* (once on call, once on return).

To fix this, we introduced the stack high-water mark.
This is configured with the range of the stack and tracks all stores into that range.
Any store below the current high-water mark (remember, stacks grow down, so the 'high'-water mark is actually at the bottom of the memory for the stack) moves the mark.
The switcher can read this before and after the call and zero only memory that's used.
If you start a thread, do some stack allocation, and then call a new compartment, we don't need to do any zeroing.
If you call some internal functions, return, and then call another compartment, we zero only the bit of the stack that you've left data on.
If you call a compartment that uses 128 bytes of stack space then we will zero 128 bytes of stack on return, independent of the stack size.

This means that we *already* had a mechanism for dynamically determining how much stack a particular compartment entry point needed.
All that we needed to do was expose it.
For C++, we've [wrapped this up in a class that lets you report the highest stack usage for a function across multiple invocations](https://cheriot.org/book/compartments.html#_ensuring_adequate_stack_space).
A lot of functions have data-dependent control flow and so data-dependent worst-case stack usage.
By running a set of tests over a function, you can find the worst-case stack usage.
This class can either log the highest stack usage that it sees or can crash the compartment if the expected amount is exceeded.

Once you've made determined the maximum stack usage for a function, you can use the `__cheriot_minimum_stack` attribute to set the value in the export table.
We've now done this for the allocator and the scheduler, which should make both robust.

Confidentiality and integrity were the primary goals for CHERIoT.
Our first adversarial security evaluation did not find any ways of violating confidentiality or integrity.
Since then, we've been working to add availability to the guarantees that we can provide.
This is just one of many steps in making the CHERIoT platform a solid foundation for high-availability embedded systems.
