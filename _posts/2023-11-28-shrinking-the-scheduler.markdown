---
layout: post
title:  "Shrinking the scheduler"
date:   2023-11-30T17:30+00:00
categories: scheduler tcb
author: "David Chisnall"
---

The CHERIoT RTOS scheduler is, by definition, in the TCB for availability.
It chooses the next thread to run and so can choose to run the wrong thread (or not run a thread).
As such, it's a good idea for it to be small.

The original version that Hongyan Xia wrote had two communication primitives intended to make it easy to port FreeRTOS code:

 - Message queues (multi-producer multi-consumer FIFOs of fixed-sized messages)
 - Event groups (bitfields that let you wait on a certain subset being ones)

This was very effective as a starting point and Hongyan was able to port some quite large components from FreeRTOS with a thin compatibility wrapper and few code changes.

The initial synchronisation and communication mechanisms were later joined by futexes (somewhat misnamed, since we don't have a 'userspace' as such).
A futex-like mechanism is now present on all major (non-embedded) operating systems and is even required for C++'s `std::atomic`, which (as of C++20) exposes [`wait`](https://en.cppreference.com/w/cpp/atomic/atomic/wait) and [`notify_one`](https://en.cppreference.com/w/cpp/atomic/atomic/notify_one) / [`notify_all`](https://en.cppreference.com/w/cpp/atomic/atomic/notify_all) methods that expose futex-like behaviour.

A futex is a very general synchronisation primitive.
The wait operation is an atomic compare-and-sleep, which returns immediately if the value in memory is not the expected value.
The notify mechanisms wakes up one or more sleepers.
This works even in a CHERI world with separate permissions.
The wait operation requires read access, the notify operation requires write access, but neither requires both.
After the initial check, the scheduler associates sleepers with an address and so doesn't need to be able to capture the pointer that's passed.

The scheduler has to be in the TCB for availability but ideally it shouldn't be for confidentiality or integrity.
For event groups, this was mostly true.
A malicious or compromised scheduler could set or clear event notifications, causing things to wake spuriously or not, but you could at least mitigate this by also communicating any key state separately from the event group and using it just to tell you to check something else.

Message queues complicated this picture.
The scheduler could see all messages in message queues.
Worse, message queues were reachable from the thread structures and so a compromise of the scheduler in one thread could leak or corrupt any data in any in-flight message queues.
This put the scheduler in the TCB for confidentiality and integrity of the contents of message queues.

Futexes can be used to implement a rich set of locks and can also be used to implement message queues and event groups.
This is precisely what [PR139](https://github.com/microsoft/cheriot-rtos/pull/139) did, removing these from the scheduler.

Message queues are interesting because they have a variety of threat models.
In some cases, they're used for passing messages between threads in the same compartment.
In others, they're expected to be a trusted intermediary between two (or more) compartments in different trust domains.

In the first case, both ends trust each other to follow the rules and so we can expose a message-queue library and invoke it without a cross-compartment call.
This makes the overhead of sending and receiving in a message queue lower.
You need a cross-compartment call into the scheduler only if another thread may be blocked on a condition that you've just changed (such as making a queue non-empty) or if you need to sleep (for example, trying to sending a message to a full queue or reading from an empty one).
All other cases are a direct call to a shared library and require no cross-compartment interaction.

In the second case, we don't want to expose the internal implementation details to either compartment.
A malicious (or buggy, or compromised) compartment that can directly manipulate the queue could modify values that are in the queue already, modify producer or consumer counters to cause the other end to trap trying to access out of the bounds of the buffer, and so on.
A compartment that exposes sealed endpoints addresses this, serving as a trusted intermediary that manages the state of the queue.
The scheduler was responsible for this, but in the new version there is an optional compartment that can fill this rôle.

This compartment is completely unprivileged and compromising it will not impact the scheduler.
It is also entirely stateless.
Two different message queues are never visible to it at the same time, all communication between threads happens via message queue objects that are passed in (as sealed capabilities) from callers.
This means that, if you find a bug in the message queue code (entirely possible!) and force it to crash, it will not affect any other message queues, only yours.
Even if one message queue caller manages to gain arbitrary-code execution in the compartment *there is no way to get access to any other message queue*.

This is the kind of security guarantee that you can build when you start with a CHERI core.

The new code also uses per-queue and per-event-group synchronisation.
The original version in the scheduler ran with interrupts disabled.
This simplified the code but meant that some operations could delay interrupts for a long time.
For example, if you created a message queue with 4 KiB elements, the scheduler would copy the entire 4 KiB with interrupts disabled, consuming several hundred cycles.
The new code runs entirely with interrupts enabled.
The only time interrupts are disabled is when the futex wait and wake functions in the scheduler are called.
These do small and bounded amounts of work.

So how does this affect the TCB size?
Before we started this refactoring, the scheduler was 9,088 bytes of code.
After removing message queues and event groups, it shrank to 4,528 bytes, just under half the original size.

This PR does one more change: it removes the `sched` namespace.
A lot of the scheduler's functionality was implemented in helper functions in the `sched` namespace.
These were only ever called in the single compilation unit for the scheduler but the compiler had to assume that they might be called elsewhere.
As a result, it didn't inline a number of calls that it should have done.
Removing this and moving these functions into an anonymous namespace shrank it down to exactly 4,000 bytes.
For those keeping score, that's 44% of the original size: a nice reduction in the amount of code that you have to trust for availability.

This also means that the scheduler is untrusted for confidentiality or integrity of anything other than futex words (which are 32-bit integers and so cannot hold pointers).
It no longer tracks any state for message queues and so a compromised scheduler can't leak the contents of message-queue messages.

A follow-on PR then updated how thread IDs are accessed, shrinking the scheduler further to 3,904 bytes.
Even combined with the message queue library and compartment, and the event group library, this now adds up to 8,304 bytes: less than the total size of the scheduler when we started.
The total code size has shrunk, the size of the TCB for availability has shrunk a lot, and now the damage that an vulnerability in the event group or message-queue code can do is severely limited.

