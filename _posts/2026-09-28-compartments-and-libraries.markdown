---
layout: post
title:  "Why does CHERIoT have compartments and libraries?"
date:   2026-09-28
categories: philosophy rtos
author: David Chisnall
---

CHERIoT provides users with two abstractions that look quite similar:
Compartments, which resemble MULTICS shared libraries, and shared libraries, which resemble a cut-down version of UNIX shared libraries.
The reason for this is something that we've learned with many examples during the CHERI project:

**All security boundaries are software-engineering boundaries but not all software-engineering boundaries are security boundaries.**

What are software engineering boundaries?
-----------------------------------------

From the advent of structured programming onwards, a big part of software engineering has been about modularity and abstraction.
Whether these components are functions, objects, modules, or libraries, the core idea is to expose them as abstractions.
Users can understand *what* a component does without having to understand precisely *how* it does the thing.

This gives two significant benefits.
It reduces the cognitive load for users of the component.
They do not need to keep track mentally of all of the internal behaviour, whether it's state or algorithms, they simply need to understand its public interfaces.
It also increases freedom for the implementers of the component.
They can replace private data structures and algorithms with different ones without needing the users of the component to change anything.
They can completely refactor their code and that doesn't matter to the caller.
Similarly, the implementer of a module doesn't need to understand the code that uses the module.


This happens even in trivial places and with simple abstractions.
For example, modern C implementations often have aggressively optimised implementations of `strlen` or `memcpy`, but users of these functions don't need to be aware of how these optimisations work.

What are security boundaries?
-----------------------------

Security boundaries much stronger because they presume the presence of an adversary.
Where a software engineering boundary says that you don't need to see across the line, a security boundary may say that you *cannot* see across the line.

This can be a boundary in one or both directions.
A sandboxed function can leak information to the caller but can't leak the caller's state to the callee.
A safebox function must be able to protect its own data but doesn't need to be protected from leaking caller state.

Importantly, all of these come with a *threat model*.
They need to understand what they're trying to protect, from whom, and what the attacker might have access to.

Not all software-engineering boundaries are security boundaries
---------------------------------------------------------------

The first place I encountered this distinction was looking at string classes.
It's a useful software-engineering abstraction to provide an abstract string type.
Users don't need to know how the character data is stored and they get a useful set of tools for things like creating substrings, and so on.

Anyone with a reference to a mutable string can mutate the data inside, so what would the threat model be?
They may even be able to alias the storage of the string.
Making a string class, or each instance of a string object, a security context would not add any meaningful benefit because the interfaces explicitly allow users to do anything.
They would add overhead for no benefit.

None of that eliminates the *software engineering* benefits of a string type.
The same is true for a lot of software-engineering abstractions.
Some are *intentionally* porous and allow you to directly manipulate internal data structures if you know what you're doing and are willing to take responsibility for bugs that you introduce when you do.

Even C APIs have problems like this.
Should `memcpy` live in a `libc` compartment, or in its own compartment?
The `memcpy` function needs at least read-only access to the source and read-write access to the destination.
The `memcpy` function doesn't own any mutable state, so it doesn't need to protect itself from a caller.
If it is a compartment of any kind, it's a sandbox.

But, on a CHERI system where pointers have bounds, what does the caller expect *might* go wrong with `memcpy`?
It can't accidentally go wrong in any meaningful way.
If you pass the wrong bounds, it will trap, but that's a trap caused by the caller.

An *actively malicious* implementation of `memcpy` might walk up the stack and arbitrarily corrupt other data.
Putting it in an isolated compartment would prevent this.
But if `memcpy` is this broken then there's very little chance that your program will work at all.

Security boundaries should be software-engineering boundaries
-------------------------------------------------------------

In the other direction, security boundaries do have a lot of the same properties as software-engineering boundaries.
They are explicit abstraction layers, where you have information hiding across a boundary.
A security boundary needs the same things that a software-engineering boundary needs, such as a clear boundary line, control over visibility across the line, and so on.

It also needs *more*.
Arguments or function returns might not be trusted and need extra validation.
The amount of implicit sharing is less across a security boundary (ideally none), whereas software-engineering boundaries can be slightly amorphous.

Software-engineering boundaries often use some form of *opaque types*.
These hide the implementation details and give callers a handle (often a raw pointer).
When you harden the interface into a security boundary, you discover that this is is a very powerful abstraction *if* you can protect those references.
This has been known for a very long time.
File descriptors in UNIX and handles on Windows have been there from the origins of their respective platforms and provide an opaque type exposed by the kernel.
The software-engineering abstraction is used in existing secure interfaces, just with additional hardening (a Windows `HANDLE` looks like a pointer, but you can't actually dereference it, for example).

A lot of effort in various platforms has gone into making their security boundaries look like software-engineering boundaries.
System calls *look like* functions.
Various remote procedure call (RPC) frameworks for communicating between privilege-separated components do the same in a more general way: make crossing a security boundary (an RPC to another process) look like a software engineering boundary (a function call).

CHERIoT's answer
----------------

In CHERIoT, *compartments* define security boundaries.
They are also software-engineering boundaries.
They expose functions, can share data by passing pointers, and can [use sealing to implement safe opaque types](https://cheriot.org/rtos/sealing/2025/11/06/sealing.html).

A compartment's interface is a software-engineering boundary that is *also* a security boundary.
The abstractions used to build security boundaries are all designed to reflect existing tools that programmers are *already* using to create clean software-engineering boundaries.
The tools that we create for programmers to use for *hardening* those boundaries (checks on capabilities, sealing, claims, and so on) are strictly additive.

At the same time, we realise that not all code reuse should be across a security boundary.
You can copy code from a source that you trust into a compartment.
This code may have existing abstractions that you don't explicitly pierce, but you don't need to make them into security boundaries.

If this were the only way of reusing code, we'd end up with a lot of duplication.
This was the original motivation for shared libraries in UNIX and other mainstream systems.
It's also the motivation for CHERIoT's shared libraries.

Our shared libraries define a software-engineering boundary, while enabling run-time code sharing.
CHERIoT shared libraries are stateless (they have no mutable globals), which means that the same code can be trivially shared on a CHERI system.
Code in them is invoked directly, you don't go via the switcher, which would clear registers and truncate the stack for a cross-compartment call.
They're fully trusted by the caller.

Our answer to 'which compartment should `memcpy` live in?' is that it lives in the `freestanding` library, a shared library that provides the functions needed for a freestanding C environment.
The same is true of a lot of other useful functionality.
And this means that it simultaneously lives in every compartment that calls `memcpy`.

By using the same abstractions in both layers, we can often trivially wrap one in the other.
The CHERIoT message-queue components are a good example of this.
If you trust the implementation to not be actively malicious and you're communicating between components (such as threads in the same compartment) that you trust to not corrupt the queue state, you can use the message-queue library.
If you want to communicate between threads in different compartments, or you simply don't trust the library not to attack your code, the message-queue compartment wraps it in an isolated security context.

As a side effect, writing code in this style often gives flow isolation for free.
The message-queue library has to be stateless because all CHERIoT libraries are.
All of the state is held in the message queue, which is passed in by pointer as a function argument.
When this is wrapped in a compartment, this becomes a (sealed) type-safe opaque type.
The compartment still has no mutable globals, so when it's operating on one message queue it has no access to any other queue's state, or any other mutable state.
Even if you can push a message into a queue that somehow gets you arbitrary-code execution in that compartment, you can't violate the confidentiality or integrity of any other queue.

Building security boundaries on top of the same abstractions that you use for building software-engineering boundaries is very powerful.
