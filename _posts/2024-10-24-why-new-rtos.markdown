---
layout: post
title:  "Why did you write a new RTOS for CHERIoT?"
date:   2024-10-24
categories: rtos philosophy history
author: "David Chisnall"
---

I'm often asked why we decided to write a new RTOS for CHERIoT instead of using something that already existed, such as ThreadX, FreeRTOS, or Zephyr.
The short answer is that CHERIoT is a hardware-software co-design project and retrofitting ground-up co-design is hard.
This post is for people who want the long answer.

Microsoft bought ThreadX (and renamed their RTOS to Azure RTOS) shortly before we were starting the CHERIoT project and so it was natural that, as a Microsoft Research project, we'd aim to use it as a base.
Unfortunately, as we looked at ThreadX we found that there was a lot of tight coupling that made compartmentalisation difficult.
If we did it, the result would look so unlike ThreadX that we wouldn't get much benefit.
It would not be compatible with existing ThreadX uses.

We also considered some other potential options but found similar issues.

## Security is not an add-on

We believe that building a secure system means enabling and respecting two key principles:

 - *The principle of least privilege* means that no component should run with privileges that it doesn't need.
 - *The principle of intentional use* means that no component should accidentally exercise privileges that it needs for some purpose other than its current operation.

CHERI is designed around these principles and CHERIoT inherits them.
The principle of least privilege is embodied in CHERI by allowing you to hand a pointer that grants access to a single byte of memory, perhaps with only read permission, if that's all that a function requires.
The principle of intentional use is embodied by CHERI because you must use the *correct* capability as the operand to load or store instructions.
Just because you hold pointers to two objects doesn't mean that an out-of-bounds access to one will let you access the other.
In contrast, memory management unit (MMU) or memory protection unit (MPU) designs may enable least privilege (though not in a way that's convenient for programmers: more on this later) but they can't enforce the principle of intentional use.
If I have access to two memory regions protected by an MPU and I do some invalid pointer arithmetic from one, I can access the other.

CHERIoT extends these principles into the software space.
I'll talk later about how we build a software capability model to authorise operations at higher levels of abstraction than 'can I read or write this object' but the principle of least privilege permeates the design.

As with microkernel designs, we do not have a single large privileged entity.
Unlike most microkernel designs, we don't even have a single *small* entity in the system that holds the rights do absolutely anything (at least, not after the bootloader has run, and the bootloader is deterministic and does not process any data except that in the firmware image).

The most privileged entity in the system is the *switcher*.
This is a few hundred instructions and we're starting to work with some folks interested in formally verifying it (let us know if you want to join in this activity!).
The main privilege that this holds is the ability to access the register that contains a capability to the register-save area for context switching threads and the trusted stack used to enforce call-return discipline on cross-compartment calls.
This is quite a rich set of permissions because it would allow the switcher to violate the compartment- and thread-isolation guarantees that the rest of the system depends on.
It isn't omnipotent though.
It can't access all memory indiscriminately.
It can't modify its own code.
It can't read or write anything that the caller or callee couldn't access in a cross-compartment call.

The core of the RTOS also contains a scheduler and an allocator.
The scheduler has direct access to devices like the timer and interrupt controller.
It chooses the next thread to run but it cannot see the state of any interrupted thread.
When you call into the scheduler for a blocking operation, it's simply another compartment and it has access only to the things that you pass it.

The allocator is somewhat more trusted.
It holds a capability to the entire heap (though not to any other compartment's memory).
It subdivides this and manages the hardware features required to prevent use-after-free.
As such, it is in the TCB for heap memory safety (both spatial and temporal).
If; however, you are using only stack and global objects, the allocator is entirely outside of your TCB.

Other parts of the RTOS are optional libraries or compartments using the same model.
This is hard to retrofit to an existing design.
Building in least-privilege principles is easy if you do it from the start (and have the underlying hardware mechanisms to cleanly express it) but it's very difficult to disentangle later.

## Building the compartment model

CHERIoT is designed with a compartmentalisation model in mind.
The hardware and software work together to enforce this model.
The model is intended to be easy to use from programming languages.

In a traditional operating system, RTOS or otherwise, if you want to communicate between trust domains you typically use something that looks like inter-process communication (IPC).
There's a kernel that mediates communication and lets you do things like sending messages or streams of data.
In the CHERIoT compartmentalisation model, you simply call functions.

If you want to share data between two security contexts then, in a traditional operating system, you create a shared memory region and configure the MMU or MPU to expose it to both security contexts.
In CHERIoT's compartmentalisation model, you pass a pointer as a function argument to a cross-compartment call (you can now also [statically share objects](https://cheriot.org/sharing/rtos/compartments/2024/08/15/sharing-objects-between-compartments.html)).

Trying to introduce these concepts to an existing RTOS involves significant redesign.

For example, FreeRTOS implements message queues in the kernel (as do most operating systems for larger systems).
In CHERIoT RTOS, we provide a similar abstraction but in several layers.
The CHERIoT RTOS scheduler is in the TCB for availability (by necessity: a scheduler is the thing that chooses the next thread to run and can violate availability by not running a thread) but not for confidentiality or integrity.
It provides a single synchronisation primitive: a futex.
A futex is a conceptually simple mechanism that allows a thread to block if a memory location contains an expected value and another thread to wake one or more waiters.
You can use a futex to build a variety of lock types and you can also use it for producer and consumer counters in a ring buffer so that producers can sleep if the queue is full and consumers can sleep if it is empty.

CHERIoT RTOS implements a message queue in a shared library.
A CHERIoT RTOS shared library has code and read-only globals but no read-write globals and does not require crossing a security boundary to invoke.
Calls to shared libraries are about as fast as calls to functions in the current compartment but the same code can be shared by many compartments.
The message-queue library avoids any cross-compartment calls on the fast path and calls the scheduler only when a producer or consumer needs to sleep (when the queue is full or empty, respectively).

Often, CHERIoT RTOS systems will use message queues for communicating between threads in the same compartment and so this is sufficient.
Alternatively, you may want to use them in a way that's closer to traditional inter-process communication, where the sender and receiver are in different compartments and a shared library with both ends having complete access to the queue would be a problem.
For this use case, we provide a compartment that owns the message queue and provides sealed capabilities as opaque handles for the endpoints.

## CHERI doesn't look like an MPU

A lot of embedded operating systems were not designed with any kind of protection in mind and had some MPU support retrofitted.
Others were designed to take advantage of an MPU.
Neither ends up with abstractions that map well to CHERI.

For example, FreeRTOS has the notion of a *restricted task*, which is similar to a process in a conventional operating system.
A restricted task has access to a limited set of MPU regions, rather than the entire address space.
This model lets you run some code with reduced privileges, but does not help with the *principle of intentional use* and so does not align well with a CHERI model.

In an MPU-based approach, you may wish a component to have access to (the memory ranges containing) objects A and B, so you would configure MPU regions that grant access to A and B.
If there is a bounds error in addressing into A then the component may accidentally overwrite B.
In contrast, in a CHERI system, you would pass bounded pointers (CHERI capabilities) to A and B.
Any operation that attempts to access A will never accidentally access B.
This intentionality (you can access an object only when you intend to access *that object*) is core to CHERI.

The CHERIoT security model was designed to support language-friendly abstractions.
You don't share objects in CHERIoT RTOS by creating shared regions of memory and populating them, you share objects by passing pointers as arguments to functions.
This comes with a rich hardware-enforced permission model that lets you express properties such as shallow or deep immutability or no-capture guarantees on pointers.

For example, the TLS compartment in the network stack maintains a per-connection ring buffer that it uses for incoming and outgoing messages.
When it wishes to receive some data, it creates a bounded pointer to a region in that buffer that has only store permission.
The TLS compartment then passes this to the TCP/IP compartment's receive function.
When this call returns, the TLS compartment knows that this buffer is safe from time-of-check-to-time-of-use problems because the TCP/IP compartment cannot have captured it because the pointer does not have global permission.
It knows that no data outside of that buffer can have been written, because the pointer is bounded.
It knows that data left over in that buffer cannot have been read (because the pointer does not have read permission) and so it is safe from information disclosure.
All of this is visible in the source code and is enforced with CHERI.

It might be possible to get the same guarantees from an MPU by invoking a kernel to set up an MPU region with store-only permission covering an object owned by the TLS stack, but this would look very different.
Trying to provide the same programmer model for both CHERI and MPU-based systems (or even those without an MPU) would require serious compromises.

MMUs and MPUs were not designed to provide abstractions for normal programmers.
MMUs were created to provide operating-system abstractions such as swapping and the ability to isolate processes or virtual machines from each other.
MPUs were designed to allow an RTOS kernel to protect itself from untrusted tasks.
They were both designed as tools for use by operating systems.

CHERI, in contrast, was designed to provide a protection model that could be directly exposed into C and higher-level programming languages.
It is designed to protect things that are visible in a language abstract machine as objects, not memory ranges.
It protects these objects by putting the permissions (and bounds and integrity guarantees) in things that are exposed to programming languages as pointers, not as entries in an indirection table.

## CHERI enables new abstractions

There are a lot of abstractions in CHERIoT RTOS that are possible only because of CHERI.
Most notably, we make a lot of use of the *sealing* mechanism from CHERI.
Sealing is an operation that takes a CHERI capability representing something that a programmer would think of as a pointer and another that conveys the authority to seal an object as a specific type and produces a sealed capability.
Once you have created a sealed capability, the only operation that you can do with it is unseal it with a permit-unseal capability that matches the type used for the sealing operation.

If you didn't understand that, don't worry, the guarantees are more important than the mechanism.
From a C programmer's perspective, this means that you can create opaque pointers that are tamper-proof and type safe.
Your compartment can expose an API that returns a sealed pointer for use as a handle.
A caller can store this just as they would any other opaque pointer but cannot dereference it.
When they pass it back (possibly indirectly, via other compartments), your compartment can unseal it and be certain that it points to a value of the correct type.

These objects can be created statically or dynamically and let you build a rich capability system at the software level.
The first place that you'll encounter this in CHERIoT RTOS is in the heap allocator.
Our `malloc` function is a compatibility wrapper around `heap_allocate`, which requires an *allocator capability* to authorise allocation.
Each allocator capability has an associated quota, so you can restrict the amount of memory that a compartment may allocate.
These quotas show up in the auditing report when you link a firmware image and so you can statically see how much memory each compartment may allocate.

Some compartments allocate memory but have no quota.
This is possible because *delegation* is intrinsic to capability systems.
For example, when you want to create a TLS connection, you have to pass two capabilities as arguments to the function that the TLS compartment exposes for this purpose.
One authorises the TLS compartment to allocate memory on the caller's behalf for all of the connection state.
The other, a *connection capability*, authorises TLS compartment to create a network connection to the specific host on behalf of the caller.

This kind of model is very hard to build without CHERI.
POSIX operating systems use file descriptors for handles, where file descriptors are indexes into a table maintained by the kernel.
This model works well when you have a single monolithic kernel that can hold the file descriptors, it works far less well if you want to provide handles to different objects provided by mutually distrusting components that can be passed (for permanent or temporary delegation) between components.

## Ubiquitous shared heap

CHERIoT was designed to provide both spatial and temporal safety, both enforced efficiently in the hardware.
As such, we can rely on a shared heap, even in situations where you need to provide mutual distrust.

Building an operating system assuming that you can safely share heap allocations between mutually distrusting components leads to some very different design choices.
There's often a lot of API complexity from having to first query how much space something needs, then allocate the space, and then pass it and the length back to another API.
This is particularly hard when the size of the available data can change between the calls.
In a CHERIoT system, this kind of API is unnecessary.
The caller provides a capability that allows allocation and then the callee returns a pointer (owned by the caller) that is the correct size.

There are a lot of places where usability and efficiency are improved by being able to rely on a (spatially and temporally safe) shared heap but this is not something that can be provided without CHERIoT (or similar CHERI extensions).

## Auditing the compartment graph

The CHERIoT ABI was designed to enable link-time auditing.
A compartment must be explicitly provided with capabilities to do anything outside of its own code and global space.
The structures for creating these capabilities are present in the firmware image and are visible to the linker, which generates a report containing:

 - The hashes of the object code that went into the compartment (for integration with SBOMs and a secure compilation flow).
 - The functions exported from each compartment and library, and whether they run with interrupts enabled or disabled.
 - The functions in other compartments or libraries that a compartment or library calls.
 - The MMIO devices that a compartment can access.
 - The software-defined capabilities that a compartment holds, their type, and their contents.
 - The stack and trusted stack sizes and entry points for all threads.

These reports are intended to be consumed by the [`cheriot-audit`](https://github.com/CHERIoT-Platform/cheriot-audit) tool, which can drive CI checks, code-signing decisions, and so on.

Supporting this required carefully designing the ABI along with the compartment model.
It required building support into the linker for report generation and having a build process that created the right-shaped inputs.
We would have lost these properties if we'd tried to fit into an existing linkage model.

## An RTOS is small

Finally, being small is one of the key selling points for an RTOS.
ThreadX advertises a code size of 2-15 KiB for most deployments.

If we had decided to rewrite something like Linux or Windows and aim for feature parity, that would have been a multi-billion-dollar effort and would have required a lot of justification of the potential benefits.
Rewriting an RTOS, on the other hand, is far more feasible.
The core parts of CHERIoT RTOS add up to around 7 KLoC.
Including headers and libraries, it's only around 20 KLoC.
This made the cost-reward calculation very different: the cost was small and the rewards of a co-designed hardware-software stack were large.

It is far more important that we can easily reuse existing code that was written either for bare metal or other embedded operating systems.
For example, our FreeRTOS compatibility layer makes it possible to run the FreeRTOS TCP/IP stack, MQTT library, SNTP library, and so on.
Out prototype network stack is around 5 KLoC for the new code, but it includes well over a hundred thousand lines of off-the-shelf third-party code.

By providing a core RTOS that has a rich compartmentalisation model, we make it easy to take existing components and wrap them in mutually-distrusting least-privilege compartments.

## So, Why a New RTOS?

CHERIoT RTOS is co-designed with its underlying architecture and its C/C++ toolchain to efficiently provide programmers with affordances that are difficult, expensive, or even impossible to achieve in embedded computing platforms that run software stacks that had to work around the limitations of existing hardware.
Try it yourself by [running CHERIoT-RTOS in a GitHub Codespace](https://codespaces.new/CHERIoT-platform/CHERIoT-rtos?quickstart=1)!

