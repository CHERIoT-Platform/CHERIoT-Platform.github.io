---
layout: post
title:  "Improved error handling in CHERIoT RTOS"
date:   2024-09-20
categories: rtos errors
author: "David Chisnall"
---

CHERI platforms in general, and CHERIoT in particular, can turn a lot of bugs that would be silent data corruption into recoverable errors.
The 'recoverable' part comes from the fact that the error is caught *before* you perform the invalid operation.

In [CheriBSD](https://www.cheribsd.org) and the proposed CHERI extensions to POSIX, CHERI faults are delivered as signals.
In CHERIoT RTOS, we have a similar mechanism.
Each compartment can define an error handler and, when a CHERI fault (or other recoverable fault such as an illegal instruction or alignment trap) occurs, this is invoked with a copy of the register file at the point that the trap occurred.
This can be used to perform low-level error recovery operations, such as skipping the faulting instruction or unwinding to the previous compartment invocation.

Custom error handlers are quite difficult to write.
In addition, the design made it impossible to run them when the fault was caused by stack exhaustion.
Although we have [tools to help avoid stack overflows](https://cheriot.org/rtos/stack/programming/2024/05/01/stack-usage.html), these still happen (particularly when incorporating third-party code) and it's nice to have a general mechanism for using them.

With a few recent PRs, we've built a much more developer-friendly mechanism for handling errors.

# Unwinding the stack

Exceptions are a common mechanism for reporting errors.
Exceptions are a form of non-local return: they transfer control flow to a function higher up the stack to report the error.

On most modern platforms, exceptions are implemented using a *table-driven unwinder*.
The exact details differ between 64-bit Windows and *NIX systems, but the underlying mechanisms are very similar.
For each function, the compiler emits a table that describes how to unwind the stack.
A generic unwinder library can read this information and pop each frame off the call stack, one at a time.
Language-specific exceptions are built on top of this.
In the unwind data, each function can define a *personality function* that uses some language-specific data to determine whether the current unwind should run cleanups or should stop for a catch.

All of this adds up to a lot of code.
The unwind library and C++ exception support are typically on the order of 2-300 KiB of code.
On top of this, the unwind metadata can add 10-20% to the size of a program.
This means that, for a lot of embedded systems, the generic unwinder plus the unwind metadata would consume all of memory, leaving no space for anything else.

For CHERIoT, we went back to older mechanisms.
Two systems implemented exceptions in a similar way: 16-bit (and 32-bit) Windows, and OpenStep.
These both used a model built on `setjmp` and `longjmp`.

These functions are defined in the C standard and provide a very low-level exception-like model.
When you first call `setjmp`, it stores some current register state in the `jmp_buf` that you pass as an argument and return zero.
If you then pass the same `jmp_buf` to `longjmp`, it will jump back to the place where `setjmp` returned.

Windows' Structured Exception Handling (SEH) and OpenStep exceptions built a linked list of exception structures that contained a `jmp_buf`.
You could jump to the top exception handler simply by calling a function that popped the top entry from this stack and called `longjmp` on it.

On Windows, this was supported by `__try` and `__except` keywords.
OpenStep implemented the equivalent functionality entirely with macros: `NS_DURING`, `NS_HANDLER`, and `NS_ENDHANDLER`.

These went out of favour on larger systems for two reasons.

First, they were not 'zero cost'.
Typically, exceptions happen in exceptional circumstances.
The `setjmp`-based model meant that there was overhead entering a `try` block, but much less overhead when throwing an exception.
Table-based models make entering a `try` block (almost) free, but make throwing an exception more expensive.

Second, as register files grew, the amount of state that needed to be stored with `setjmp` increased, so the non-zero cost became an increasingly large cost.

CHERIoT is based on RV32E, which defines only 15 registers.
Of these, most are temporary and so our `jmp_buf` needs to store only four: The two callee-save registers, the stack pointer, and the return address.
This means that it takes only 32 bytes of space and requires four instructions to write to.
Our [`setjmp` implementation](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/e4ebccc60840de78a919c42e6dd7045c99b68245/sdk/include/setjmp.h#L43) is only six instructions as a result (as is longjmp).

# Providing a thread-local unwind handler

The unwinding mechanism described above depends on two things: `setjmp` and somewhere to stash a linked list of `jmp_buf`s.
The first bit is easy, but the second bit is more complex.

CHERIoT uses the local / global mechanism from CHERI to enforce strong thread isolation.
Pointers derived from the stack pointer and return addresses may be stored only on the stack.
This means that at least two of the four registers that must be stored in a `jmp_buf` by `setjmp` can be stored only on the stack.

The traditional approach of keeping the head of the linked list in a global is therefore impossible.
The slightly more modern approach of using thread-local storage (TLS) would also not work because we do not have thread-local storage.

The obvious solution is therefore to add TLS, but what would that mean?
Consider a single thread that starts in compartment A, then calls into compartment B, which calls back into A.
If we have conventional TLS, A could stash a pointer into the part of the stack that it passes to B in TLS and then in the second invocation it would be able to violate compartment isolation (reading and writing all of the on-stack state for B).
If we built a linked-list of `jmp_buf`s there, then it would be possible for the second invocation of A to jump back to the first, bypassing B and violating the trusted-stack guarantees.
This would be bad.

What we want is not *thread-local* storage but *compartment-invocation-local* storage.
Each time a thread enters a new compartment, you should get some storage that is not tied to the depth in the control stack.

To implement this, we looked at the earliest way that operating systems have implemented thread-local storage: reserve some space at the top of the stack.

Now, on entry into a compartment, the switcher will move the stack pointer 16 bytes down before transferring control into the callee.
Similarly, the loader will reserve 16 bytes at the top of the stack before starting a thread.

This means that you have two pointers worth of space that are easy to find (set the address of the stack pointer to its top, then set the address to eight or 16 bytes below that).
CHERIoT has a convenient `cgettop` instruction and so this sequence is very short.  First, `cgettop` gives the top address, then `csetaddress` gives a new capability derived from the stack pointer that points to the address.  After that, a `-8` immediate offset to a load or store capability instruction can access the space, so we need only three instructions to load the head of the list.

With this, we can store the head of the linked list of error handlers at the top of the stack.
When you want to jump to the nearest error handler, find the head of the list relative to the stack pointer, pop it, and pass it to `longjmp`.
The `cleanup_unwind` does all of this for you, so typically you won't need to ever see that this is how it's implemented.

# Handling errors even in the presence of stack overflow

Everything so far is enough to build a set of nested error handlers, but what happens if the error is caused by running out of stack space?
[PR 301](https://github.com/CHERIoT-Platform/cheriot-rtos/pull/301) adds the last bit of this: support for a new kind of error handler that doesn't use a stack.

This is in addition to the existing error handlers.
The stackless error handler (if it exists) will run instead, in any of the following conditions:

 - A compartment doesn't provide the normal error handler.
 - There isn't enough stack space available for the context.
 - The stack isn't valid at all (the stack pointer can become untagged if a function prologue subtracts from it and moves it out of bounds).

This means that you can now run an error handler even if you overflow the stack.
But what happens if you want to jump to the nearest error handler registered as described earlier and the stack pointer isn't valid?

First, the CPU will trap because a `csp`-relative load fails because `csp` (the capability stack pointer register) is untagged.
This transitions to the switcher.
The switcher will then find that you need to run the stackless error handler (because `csp` is untagged).
The switcher will then look at the trusted stack and rederive the `csp` value that you had on entry.

The stackless error handler *does* have a stack, but it doesn't have a *stack frame*.
When it's invoked, it can guarantee that `csp` is a capability that authorises access to the stack, but it can't guarantee anything about the address of that capability (other than that it will be in bounds).

That's absolutely fine for popping the top error handler and jumping to it though.

# Putting it all together

Most users will never need to know anything about the above.
They can simply use the macros or wrapper functions, and the error handler, that we provide.
The error handler is added to a compartment by adding the following line under the `compartment` declaration in `xmake.lua`:

```lua
    add_deps("unwind_error_handler")
```

C programmers have to use the macro-based version, which looks like this:

```c
CHERIOT_DURING
{
    // Do some things that may cause a 
}
CHERIOT_HANDLER
{
    // Handle the failure
}
CHERIOT_END_HANDLER
```

Note that, because these are implemented with `setjmp`, the usual rules about `setjmp` apply.
Specifically, anything that's used in both blocks must be declared `volatile`.

In C++, you can use the `on_error` function, which takes two lambdas (or other callable objects), representing the try and error paths.
If you're using RAII for cleanup, you can often omit the second.
For example:

```c++
    LockGuard g(flagLock);
    // No handler.  g's destructor runs after on_error returns.
    on_error([&]() { /* code that might fail here */ });
```

This will run the code in the lambda and then release the lock, even if you take a CHERI trap.

Hopefully this is much easier than writing a custom error handler that detected held locks and released them.

This isn't a full exception model.
In particular, there's currently no way in the handler of seeing the cause of the fault.
In a future iteration, we'll add something like 'Herbceptions': Herb Sutter's proposal for C++ where exceptions must be preallocated objects or primitive values.

This mechanism is designed to be simple and lightweight, not to be as generic as something that you'd build on a complex system.
You can protect something against faults with a single C++ wrapper function and 40 bytes of stack space.
On a system where stacks are often under 1 KiB and where total memory is usually much less than 1 MiB, that's feasible overhead.
