---
layout: post
title:  "CHERI Myths: CHERI is incompatible with safety-critical systems"
date:   2024-11-25
categories: cheri myths
author: David Chisnall
---

A few times, especially in the context of the Digital Catapult Technology Access Programmes, I've heard people claim that the fact that CHERI traps on memory-safety errors means that it can't be used in safety-critical systems.
I understand some of the rationale behind these claims, but the claims themselves seem the result of misunderstanding the current baseline.
The claims are usually of the form 'our code may contain memory-safety bugs but they don't impact the safety-critical parts and so it's better to leave them'.
This is a dangerous view, as I will explain below.

## Why are memory-safety errors bad?

Before we go into detail about what CHERI gives and why this may or may not be problematic, it's worth reconsidering why memory safety bugs are bad.
We often quote the Microsoft and Google numbers that 70% of vulnerabilities come from memory-safety bugs but if you dig deeper you'll see that the severity of memory-safety bugs is often disproportionately high.
The reason for this is the 'spooky action at a distance' problem.

If you have a typical logic error, you may go down the wrong branch in your code.
For example, some of the most famous early UNIX bugs involved the following pattern, which compilers will now warn about:

```c
if (uid = 0)
{
    // Root is allowed to do this
}
```

The programmer thought that they were checking the user ID was the root user, but in fact they were assigning the root user ID to the `uid` variable.
This kind of bug is often a critical vulnerability because it can directly lead to privilege elevation.
You can tell, because you can look at where the `uid` variable is used later and follow the control flow throughout the source code of the program.

With a memory-safety bug, you cannot do this.
If you write outside the bounds of a buffer, or through a dangling pointer, then you have stepped outside of the language's abstract machine.
The compiler will assume that this cannot happen when optimising and so may:

 - Reorder loads and stores, for example by pulling your memory-safety bug out of a loop.
 - Assume that the object is live if you unconditionally store to it and so elide other null checks.
 - Assume that stores through a pointer to an object are safe to forward to loads from the same object.
 - Assume that stores to one object will not affect the contents of another.
 - And so on.

If you violate the spatial and temporal memory rules of C or C++ then you are doing something that is *undefined behaviour* in the language.
The compiler does not (contrary to popular belief) hate you and use undefined behaviour to try to pick on you, it uses undefined behaviour to make assumptions about things that cannot be guaranteed by the language (or, at least, by common implementations).
If a function takes a `char*` as an argument and accesses `char` 5 from that index, the compiler assumes that this is within the bounds of the object that is pointed to.
Statically verifying that this is true is impossible in the general case (consider `malloc(rand())` as a trivial example - in practice it may require dataflow analysis through the heap, which is at least not feasible) and is incompatible with separate compilation.

*Any program that contains such a bug will be optimised using transforms that are not sound and so may do anything*.

This problem is the key thing that differentiates 'safe' and 'unsafe' languages.
The optimisers are often the same.
Both rustc and clang use the same LLVM optimisations and both the C and Ada front ends to GCC use the same GCC optimisations.
The safe languages provide checks in the type system that aim to prevent them from emitting things that would make transforms that the compiler will perform unsound.

This unsoundness isn't just limited to optimisation, it applies even in the lowering to machine instructions.
That store of a byte via a `char*` will be lowered to a store instruction.
If it's an in-bounds store to something that the language implementation decides is an object, the behaviour can be mapped back into something in the source language.
If it is an out-of-bounds store, or a store through a dangling pointer, then the instruction will overwrite *something*, but you can't guarantee what that will be.

In the undefined-behaviour case, this is permitted to do absolutely anything.
On conventional embedded systems, this may even include writing to a memory-mapped device register, so a single out-of-bounds write somewhere completely unrelated to device control may move an actuator connected to the physical part of a cyber-physical system.

This is why memory-safety bugs are so bad: they can do *anything*.

## What about MISRA C?

MISRA C rule 1.3 says that a program may not contain any occurrences of undefined behaviour.
As such, any program that is compliant with MISRA C will never trigger a CHERI trap.
That's somewhat facetious, but it's mostly true.
MISRA C doesn't stop there, it provides a load of other rules that help to avoid these problems.

MISRA C is a fairly restrictive subset of C that tries to confine programmers to features that are amenable to static analysis (as well as avoiding other things that cause common bugs).
The caveat with MISRA C is the same as that for Rust.
Just as Rust allows you to write `unsafe`, MISRA C codebases allow you to document exemptions from rules.
We found an instance of this in the FreeRTOS TCP/IP stack where a warning about rule 17.4 (return values must be used) was explicitly silenced.
This was in an early initialisation function that would return an error code if allocation failed two calls lower down, which resulted in a global not being initialised.
In normal use, this always succeeded because it happened during early boot, but when we introduced network stack restart then it could happen later and might fail.

For the most part, MISRA C code should not trap with CHERI, and if it does then you probably haven't actually followed the MISRA C requirements.

The same is true of Rust.
If you're using purely safe Rust, your code will never trap on a CHERI system.
If you're using `unsafe` in small, locally scoped places, you can provide guarantees that are not visible in the source code to ensure the same properties.
If you write your entire program in unsafe Rust (no one does this!) then you should expect traps.

So do I need CHERI if I'm using MISRA C or Rust?
[I've answered that before](/cheri/myths/2024/08/28/cheri-myths-safe-languages.html), but broadly yes.
CHERI lets you enforce properties even in the unsafe dialects of safe languages, in C and assembly code linked with them, and on binary-only components or components compiled with an untrusted compiler (or code that may be written specifically to exploit bugs in a trusted compiler).

## What is your baseline?

To answer the complaint about CHERI trapping, it's worth considering what happens if you have a memory-safety bug today.
You will see one of the following:

 - It's deterministically benign, reads some constant or write over something that's never read.
 - It's deterministically out of a mapped / accessible region will be caught by your MMU / MPU and trap (so we are not worse)
 - It's deterministically leaking information or corrupting other state.
 - It's data dependent and so will nondeterministically do one of the above three options.

CHERI makes the first case worse if it's triggered in production, because we catch it and trap but, because errors in this category are deterministic, then it make things better because you can catch it in testing.
This is part of the 'shift left' approach: bugs caught earlier are cheaper to fix.
If you have a deterministic but 'benign' memory-safety bug, then CHERI will catch it.

The problem with the framing of a 'benign' memory-safety bug is that the benignity is a property of a specific compilation.
Any change to the compiler, or the global memory layout, may turn them from benign to malign.
Modern compilers support reproducible builds, but unless you're using all of the options to enable them then even recompiling the same source with the same compiler may trigger these issues.

Memory safety bugs exist outside of the language abstract machine and so may do a different non-deterministic thing.
Moving to a different SoC with a different memory layout and a different set of instructions may change these benign bugs into data corruption that affects safety-critical operation.
It is absolutely not acceptable to claim that you have a safety-critical system if that safety depends on behaviour that is not specified.

In the second case, the only change is the trap reason and the fact that the trap on a CHERI system gives you more information.

In the third case, the failure won't appear to be memory safety today, it will be some random data corruption.
As I wrote above, this means that you're in the 'spooky action at a distance' world.
There is absolutely no guarantee that the corrupted data is not even more safety critical.
For example, an out-of-bounds store that overwrites the dose value in an insulin pump will not crash on a non-CHERI system, but might well kill the patient.
Again, if your 'safety' guarantees are framed as 'something bad happens and we don't know what, but it hasn't killed anyone yet' then you do not have safety.

In the fourth case, CHERI improves things because now it should show up in testing reliably and you can fix the issue.

The baseline is not better in *any* of these cases.
The example that people often use here is anti-lock braking systems (ABS).
You absolutely do not want the controller for ABS to crash.
But will it if you run it on a CHERI system?

If it's written in MISRA C and passing the checks (and not arbitrarily silencing them), it won't.
If it's written in safe Rust, it won't.
If it's written in SPARK Ada, it won't.

Something like this will not be dynamically allocating memory, it will not be accessing arbitrary memory locations via untrusted pointers.
It will have a set of inputs that should be amenable to exhaustive testing.

If you have a chip dedicated to running the ABS and that code is written in a safe language, CHERI doesn't improve on the state of the art: you have already statically proven that nothing that CHERI dynamically prevents can happen.
If you are building a *mixed criticality* system, the benefits of CHERI are far more pronounced: CHERI will allow you to collocate things on the same chip that previously needed to be isolated.

## What does CHERIoT do with traps?

CHERIoT RTOS provides three ways for a compartment to handle CHERI failures (and other synchronous trap, such as invalid instruction errors):

 - If there is enough stack space, a compartment may provide an error handler that runs with a copy of the register file and can inspect and modify the state.
 - If there is not enough stack space or the first kind of error handler is not provided, the compartment may provide a *stackless* error handler.
   This is used to implement our structured error handling mechanisms.
 - If neither is provided, or the error handler returns the force-unwind result, the current compartment call is terminated and control returns to the caller with an error code in the return register.

For stateless compartments, the last is often the right approach: if something goes wrong, let the caller know.
This is often the best fit approach for safety-critical systems, where things should *never* go wrong and you've got static analysis receipts to demonstrate this.

If you're relying on the notion of a 'benign' error, the first kind of error handler helps you emulate the current behaviour (though I don't recommend it if you actually care about correctness!).
You can decode the current instruction and determine whether it's a load or store.
If it's a store, you can skip it, if it's a load you can also put a zero in the target register.
Ideally, you'd also write some telemetry that would let you fix the bug.
This turns your 'benign' memory-safety bugs into ones that have deterministic behaviour that is guaranteed not to impact the rest of the system.

The middle category is most interesting.
This form is most commonly used with `longjmp`, to jump to the nearest error handler on the stack, which would be in violation of MISRA C 21.4, which says that `setjmp.h` may not be used and the functions in it may not be used.
It's worth understanding *why* that rule exists in MISRA C.
It's not an arbitrary rule, in general `longjmp` is a terrible idea for two reasons.

First, it's hard in the general case to prevent reusing jump buffers.
Remember, MISRA C prefers to eliminate things that make static analysis hard and, in the general case, the kinds of undefined behaviour that reusing a jump buffer can trigger are impossible to reason about.
This is not a concern for our use, because we maintain a structured programming discipline for these buffers: they're pushed onto a stack when you enter a block and popped as you leave.

Second, and more importantly, they are non-local control flow.
This makes reasoning about them very hard: any function call in between a `setjmp` call and its corresponding `longjmp` may fail to return and instead resume from the `setjmp`.
This is a very good reason for avoiding these calls.
In our use, we trigger the `longjmp` (technically, not a `longjmp` call, but something equivalent) only when it would not be safe to continue to the next instruction: when we've already reached a point in the program where executing the faulting instruction would be undefined behaviour.

There's a third reason that doesn't apply on CHERIoT.
On most platforms, `longjmp` does not interoperate with C++ exceptions and other languages that use the same mechanism and so will prevent destructors from running.
We do not support other unwind mechanisms because stack unwinders are very large.

We recommend using this error-handling mechanism in one of two ways.
If you use it at very fine granularity (for example, around a `memcpy` call) then the non-local reasoning problems don't arise because you do not have deep call stacks.

If you use it at a very coarse granularity (for example, in the function entry point) then the non-local reasoning problem doesn't apply because your handlers are big hammers.
For example, in the TCP/IP compartment of the network stack, the error handlers free all memory owned by the compartment, push all blocking calls out with an error return, restore all globals to known-good state, and restart the TCP/IP stack.

Both of these would merit MISRA C exemptions, but if the rest of the code in the compartment is fully compliant MISRA C then they're not necessary.

Some high assurance systems reset the entire core after a few tens of thousands of cycles to ensure that it's always within a known valid space.
CHERIoT lets you do this at a compartment granularity.
In fact, some compartments work this way automatically.
In [Phil Day's configuration broker example](https://cheriot.org/security/philosophy/2024/07/30/configuration-management.html), the validators are stateless and so are implicitly reset on every call.
This kind of scoped reset is possible because CHERI catches memory-safety bugs *before* they do any damage and so we can provide strong guarantees about what is happening.

CHERI in general and CHERIoT in particular give you a lot of tools for building high-availability and mixed-integrity systems.
How you use them is up to you.
