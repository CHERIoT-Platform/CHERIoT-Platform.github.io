---
layout: post
title:  "CHERI Myths: Writing C/C++ for CHERI is hard"
date:   2024-08-22
categories: cheri myths
author: David Chisnall
---

I've had several conversations over the past six months where people who have never written C/C++ code on CHERI have told me that they expect it to be harder than on non-CHERI systems.
I struggle a bit to understand this.
If it were true, using tools like [valgrind](https://valgrind.org) and [Address Sanitier](https://github.com/google/sanitizers/wiki/AddressSanitizer) would make development harder, which makes you wonder why these tools exist.

I recently wrote some C and C++ for a non-CHERI target and, honestly, I can't believe I used to do that regularly given how much harder it is.
Even in environments with a fully working interactive debugger, writing working C/C++ is more effort than on CHERIoT where we don't (yet) have debugger support.

Imagine you have an off-by-one error that overflows a buffer.
On non-CHERI systems, it's hard to track down.
On the stack, it may be in padding and have no effect.
It may have no effect in debug builds, but cause corruption in release builds where the stack layout is different.
If it's in the heap, it may corrupt some unrelated object and the symptoms show up much later.
I run in valgrind or address sanitiser and hopefully get a useful result.

On any CHERI target, I get a deterministic fault.
Every time I read or write that one-byte-out-of-bounds value, I get the same fault.
On [CheriBSD](https://www.cheribsd.org), I'd attach the debugger and see where it happened.
On CHERIoT, until we get a working debugger, I'd include the (somewhat poorly named) [`fail-simulator-on-error.h`](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/main/sdk/include/fail-simulator-on-error.h) header, which installs a default error handler.
When the error is triggered, this prints the exact instruction that tried to read or write out of bounds.
I'd then look in the dump file, which would tell me the line number, and fix it.
This typically takes me a minute or two, if that.

Similarly, if I have a use-after-free error, there's some probability that address sanitiser will find it.
Valgrind is a bit better, but is *very* slow.
On CHERIoT, I get a trap as soon as I try to use the dangling pointer and I fix it in the same way as a spatial error.

Importantly, the CHERI exception happens *before* any data corruption.
I'm not trying to work backwards from a point where my heap or stack is corrupted to try to find the place where the corruption occurred, I'm told exactly where the bug is.
The first use of a dangling pointer or the first out-of-bounds access to an object will trigger a CHERI exception and point to precisely the instruction that is doing the wrong thing.

Note that all of this is about *incorrect* code.
CHERI C and C++ try very hard to give you a standards-compliant (and de-facto standards-compliant, allowing things that the standard leaves open to implementations but everyone assumes are fine) implementation.
Almost all of the C and C++ code that we've tried to run on CHERIoT has worked with no source-code modifications.
Most of these are well-tested codebases, sometimes MISRA C with loads of static analyses run, which *probably* don't have any memory-safety bugs.

The things that cause CHERI traps are undefined behaviour in C/C++.
When your program does something that is undefined behaviour, the space of possible behaviours is unbounded.
You may get a segmentation fault.
You may get arbitrary data corruption.
You may get a totally unexpected sequence of instructions executed.
Bugs that introduce undefined behaviour are the *hardest* to debug, because they mean that later code (or, in some exciting examples, [earlier code](https://devblogs.microsoft.com/oldnewthing/20140627-00/?p=633)) is all depending on properties that are not true and so can do absolutely anything.
Trapping on these things, rather than corrupting state, is a *huge* improvement to the debugging experience.

If you're writing correct code, you probably won't notice the difference between CHERI and non-CHERI systems.
If you're writing buggy code (which, let's face it, we all do, at least some of the time), CHERI lets you catch errors sooner.

We've heard from several of the companies that prototyped on [Morello](https://www.morello-project.org) that they want to keep their Morello systems for CI for precisely this reason: testing in Morello finds bugs earlier.

The 'shift-left' idea comes from the fact that bugs cost more the later they're found.
If you can avoid bugs at the design time, that's perfect.
If you can avoid them before you ship a product, that's good.
If you can detect them in production and recover, that's okay.
If you don't detect them and they impact customers, that's the worst (just ask CrowdStrike).
Developing for a CHERI target makes it easy to find bugs before you ship them.
It typically costs at least one order of magnitude less to fix them at this point than after deployment.

The 'shift-left' benefits for CHERIoT don't end at catching bugs early.
If you compartmentalise your software, failures in production can become *recoverable* failures in production.
For example, the CHERIoT network stack now [restarts the compartment that contains the FreeRTOS TCP/IP stack if it crashes](https://github.com/CHERIoT-Platform/network-stack/pull/27).
From the perspective of the rest of the system, all connections drop (something that you have to handle anyway because networks are unreliable) and need to be reconnected.

All of this makes developing and shipping products cheaper on CHERI systems than on conventional hardware.
