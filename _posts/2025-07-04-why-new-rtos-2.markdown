---
layout: post
title:  "Why did you write a new RTOS for CHERIoT? (Part 2)"
date:   2025-07-04
categories: rtos philosophy history
author: "David Chisnall"
---

Back in October last year, I wrote a bit about [why we wrote a new RTOS for CHERIoT](/rtos/philosophy/history/2024/10/24/why-new-rtos.html).
Reading that again, I realise that it had a lot of high-level concepts but missed out on some detail.
This time, I wanted to take a closer look at some CHERIoT RTOS features to show that being able to rely on CHERI lets us build them in fundamentally different ways to other systems.

Message queues
--------------

On an operating system with a monolithic kernel, message queues are typically provided by the kernel.
You do a system call and that copies some data from userspace into a kernel buffer, or blocks if the buffer is full.
Receiving looks similar, copying out of the kernel buffer or blocking if the buffer is empty.

This is the approach taken on [Linux](https://manpage.me/index.cgi?apropos=0&q=mq_overview&sektion=0&manpath=CentOS+7.1&arch=default&format=html), [FreeBSD](https://man.freebsd.org/cgi/man.cgi?query=mq_open), and even [FreeRTOS](https://www.freertos.org/Documentation/02-Kernel/02-Kernel-features/02-Queues-mutexes-and-semaphores/01-Queues) and [Zephyr](https://docs.zephyrproject.org/latest/kernel/services/data_passing/message_queues.html).
There are good reasons for this approach.
When you transition into the kernel, you are in a system that has access to the current userspace process's memory and so this copy is easy.

There's an obvious down side to this from the perspective of secure system design: The kernel has read and write access to all memory owned by user processes.

Adapting this to a microkernel design is harder.
Microkernels try to provide a small kernel (which still typically has access to all memory), but they move most services out of the microkernel into separate, isolated tasks.

Typically, microkernels expose a message-passing model for IPC.
[seL4, for example, provides a synchronous message send](https://docs.sel4.systems/Tutorials/ipc.html), where the sender provides some data, the receiver provides a receive buffer, and the kernel can copy between them.
If you want to send a large amount of data you will send a (software) capability to a page that you want to grant access to.
The kernel will add this capability to the receiver's capability tables, allowing the receiver to map the memory.

If you want to build a message queue with buffering on seL4, you expose a new task that owns the buffers and allows the kernel to copy in and out of them in response to send and receive events.
Or, for large messages, you use seL4's page-granularity capability model to temporarily allow the message-queue task to have access to some of the sender and receivers' pages to do single-copy transfers.
If you want to isolate different buffers (to protect yourself from bugs in the message queue logic: remember, this code is an addition that is *not* part of the formally verified core of seL4) then you need a new instance of this task for each message queue.

Embedded systems could do a bit better in some ways because MPUs tend to allow finer granularity than MMUs.
An embedded microkernel could grant read access to the source range to the message-queue service as part of an IPC message.
The message-queue service could then copy into a buffer that it owned and the kernel could then remove that permission at the end of the call.

There are some problems to do this securely:

 - MPUs typically have limited numbers of active regions (16 is common).  Adding more is painful for power and area because they're an associative lookup that's consulted on every memory access (including instruction fetch).
 - The message-queue service needs to be careful that it's really copying from the newly mapped region.
   If the region is adjacent to some other region that the service has access to, an overflow could cause problems.
 - If you wanted to further privilege-separate the system, delegating access to the caller's buffer *and* the message queue (or one element in the queue) adds more complexity.
 - The message queue still needs to interact with the scheduler for blocking, which interacts with timeouts in complex ways.

All of this comes back to my refrain for the CHERI project:

<p style="text-align: center; font-weight: bold">Isolation is easy, (safe) sharing is hard.</p>

An MPU or MMU allows you to isolate processes or tasks very easily, but this kind of privilege-separated design relies on being able to easily share memory.

On CHERIoT, this is easy.
When you call into the message-queue compartment to send or receive a message queue, it's a normal cross-compartment call.
This takes a capability with read (for send) or write (for receive) permission and does not require global.
The sharing is implicit: you pass a pointer as a function argument, the receiver can use the pointer.
The hardware and switcher provide the following guarantees:

 - If you removed store permission when sending, the message-queue compartment cannot modify your copy of the data.
   Even if that compartment is malicious, your copy is safe.
 - If you removed load permission when sending, the message-queue compartment cannot read stale data from the receive buffer.
   Even if that compartment is malicious, it cannot violate confidentiality (except for message queue contents, though you can avoid even this by passing sealed capabilities in the queue).
 - If you removed global permission (on-stack objects start life without this) then the message queue cannot have captured a copy of the pointer, so once the call returns you know that there's no additional aliasing.

These are foundational guarantees in CHERIoT but they make implementing something like a message queue that needs to share buffers with callers and callees very easy.

There's one more aspect to implementing something like a message queue: How do you identify one?
On POSIX systems, this is via a *file descriptor*, an `int` that refers to an entry in a per-process table.
Passing one of these between processes requires passing them over UNIX domain sockets, where they will receive a different (process-local) number at the far end.

On CHERIoT RTOS, if you want to have a message queue that you can safely use between compartments, the type is `struct MessageQueue *__sealed_capability`.
This looks just like a pointer.
You can pass it between compartments: You can pass the write end of a message queue to a compartment that you want to be able to send messages, you don't need complex out-of-bound messaging.

The CHERIoT message queue compartment has no mutable globals.
This gives **flow isolation** for free.
Each call into the this compartment is isolated from any other thread that calls into the same compartment, if that thread passes a (sealed) pointer to a different message queue.
Even if an attacker somehow manages to gain arbitrary-code execution in that compartment, the worst that they can do is corrupt *their own message queue*.
We use the same property in the TLS compartment to ensure that one TLS connection can't leak another's state.

Sealing is an extensible abstraction.
If you want to add new kinds of file descriptor to a conventional kernel, you need to add a kernel module that provides an implementation of an abstract data type inside the kernel.
To do the same with CHERIoT, you just seal with a different sealing key.
Sealing gives you *type safety* for opaque types passed between mutually distrusting compartments.
This lets you use a model that was created for clean software engineering (opaque types) as a foundation for building secure systems.

Waiting for interrupts
----------------------

Embedded systems often run code from interrupt service routines (ISRs).
This is a problem for security because ISRs run with elevated privilege (they block interrupt delivery and they may see register contents for interrupted code).
They're also not a great programmer model because code in ISRs can't yield and so can't acquire most kinds of lock (if the thread owning the lock was running, it would deadlock).

ThreadX encouraged programmers to think about lightweight threads instead of writing code directly in ISRs.
This is the model followed by most big operating systems as well: code in ISRs should wake a thread which actually handles the event.

In CHERIoT RTOS, we've adopted this model but taken it further by unifying the mechanism that we use to block waiting for events.
CHERIoT RTOS' scheduler exposes a single primitive for waiting for events: A futex.
This is a compare-and-wait operation, where you provide a 32-bit value and a pointer to such a value, and if the location in memory matches the expected value then you block until another thread does a `futex_wake` on that address.
Linux uses futexes extensively because they're a great building block for other things.
CHERIoT RTOS builds ticket locks, flag locks, priority-inheriting flag locks, counting semaphores, message queues, and so on with this primitive.
These all have a fast path that just requires an atomic operation and slow paths that call into the scheduler if they need to block or wake waiting threads.

The scheduler's blocking APIs (wait for timeout, wait for a futex, wait for more than one futex) all take a pointer to a `Timeout` structure.
This records the amount of time that a thread may spend blocking (along with the amount of time it has spent blocking).
These are intended to be passed through multiple compartments, so a complex operating that does multiple calls that might block can reuse the timeout pointer from the caller and stop if the timeout is exceeded.
This makes it easy to build rich APIs that have bounded blocking times.
You wouldn't build a system like this unless sharing a two-word structure across security boundaries is easy.

CHERIoT RTOS' scheduler also exposes each interrupt as a futex word, which is incremented each time the interrupt fires.
Users can see if an interrupt has fired by simply checking if this word has increased since they last processed an event (for a lightweight polling mode) and can block using the same `futex_wait` call as for software events (to return to interrupt-driven mode).
You can use the RTOS' multiwaiter APIs to wait for any of a set of futexes to be woken, which lets you wait for hardware and software events easily in the same call.

This is easy on CHERIoT because handing out a read-only view of a 32-bit word is just a matter of handing a compartment a pointer to that word.
It would be possible with MPU-based systems but would require one MPU region per interrupt that a task wants to be able to wait for (remember, most MPUs are limited to 16 or fewer regions).
You could make all interrupt counters a single readable region but then you'd have to add another mechanism if you wanted to limit which threads can wait for which interrupts.
The same is true for MMUs, except per-interrupt permissions would require a single page (4-64KiB) per interrupt.

Allocating memory
-----------------

CHERIoT provides a memory-safe shared heap.
Dynamic memory allocation became popular in most systems four or five decades ago as a way of reducing overall resource requirements.
If you have two workloads that need 64 KiB to 2 MiB of RAM then static provisioning would require you to have 4 MiB of RAM.
If they can dynamically allocate memory and they don't both hit their peaks at the same time then you may need as little as 2,122 KiB.
Practically, you may need 3 MiB, but certainly less than 4 MiB.
The more tasks you have, the less likely it is that they will hit peak memory requirements at the same time and so the savings are often larger.

The biggest risk with dynamic allocation is that it makes use-after-free bugs possible.
If you never free memory, you can't use it after you free it, so these problems are impossible on systems with no dynamic allocation.
Operating systems for big systems mitigate this risk by having (at least) two layers of allocators:

 - The operating system allocates pages to processes.
 - An in-process allocator allocates objects to the rest of the program.

This two-layer approach has the benefit that use-after-free bugs in one process can't impact another process because every process's pointers refer to a private address space.
On MPU-based embedded systems, you can approximate this by providing a heap memory range that can grow or shrink, but it's very hard to design an efficient shared heap where each consumer's allocations are contiguous in memory.

Most importantly, sharing heap objects between tasks, processes, or some equivalent is very hard with an MMU or MPU.
With an MMU, you can't share anything smaller than a page (and a lot of objects are much smaller than a page).
With an MPU, you need to update MPU mappings for each task that has access to an object, which can quickly exhaust the number of regions that you have.
This can work for sharing a single object, but what if you want to share an entire object *graph*?
You'd need to walk the graph and find every object reachable from the root, then configure shared regions for these, which is even more likely to exhaust the set of available regions.

In a CHERIoT system, sharing memory is natural.
You share an object by passing a pointer to that object to another compartment.
The caller can restrict the pointer to provide deep or shallow read-only and no-capture guarantees.
The callee can *claim* the object to prevent the caller from deallocating it in the middle of an operation (the caller can deallocate it, and the object will be removed from their quota immediately, it just won't go away immediately).
When an object is freed, the hardware ensures that all pointers to it are deleted before the memory is reused (so a use-after-free bug will trap).

This means that dynamically allocating and then sharing memory is both safe and *a natural programmer abstraction*.
This, in turn, changes how a lot of operating system services are designed.
In particular, all memory is allocated against some quota, encapsulated in an object passed by a sealed capability.
This means that it's easy to limit the amount of memory that a compartment can allocate, but also means that you can *delegate* the ability to allocate memory on someone else's behalf.

Going back to the message queue example: the message queue compartment does not have its own quota.
When you create a message queue, you pass in an allocation capability that encapsulates the quota against which the memory will be accounted.
When you free the message queue, you pass in the same capability.

When you can allocate and share memory at an object granularity, including handing out type-safe opaque pointers to specific kinds of object.
All of this means that your APIs for OS services look a lot more like conventional library APIs that *don't* have a security boundary, only now they're secure.

And that's really the key reason for building a new RTOS.
It's not just about making a secure system.
It's easy to build a secure OS if you're willing to sacrifice usability.
Our goal was to build **a platform that made it easy for other people to build secure systems on top**.
And that meant focusing on secure, reusable, programmer-friendly abstractions from the start.
That's not something that you can retrofit.

