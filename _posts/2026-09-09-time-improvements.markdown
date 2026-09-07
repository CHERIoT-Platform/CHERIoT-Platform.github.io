---
layout: post
title:  "Summertime and the refactoring is easy"
date:   2026-09-09
categories: time rtos
author: David Chisnall
---

It's been a long time since the last post here but that doesn't mean that the project has been idle.
Over the last few months, we've added quite a lot of improvements to how CHERIoT RTOS handles time.
Read on for how we've added the new features without breaking backwards compatibility.

# You're an absolute timeout

CHERIoT RTOS originally adopted the FreeRTOS notion of timeouts for blocking events.
Specifically, the scheduler counted time in ticks (scheduling quanta) and you could limit the number of ticks that a callee would sleep for.
We built an API around this model that was easy to compose.
Blocking operations took timeouts that were pointers to a structure containing two fields.
The first holds the number of ticks that had elapsed so far in blocking operations using this timeout.
The second holds the number of ticks still allowed for blocking operations.
This let you see how long a callee had slept for and you could simply forward this pointer to other blocking operations to accumulate time.

This works well for a lot of code and is a useful abstraction when you think about sleeping as *yielding*: allowing other threads to run.
A tick is a configuration-specific measure that is intended to represent the time in which a thread can do a useful amount of work.

Sometimes you want to wait a more precise amount of time.
In particular, you also want to be able to *wake* at a specific time.
For example, imagine you're driving an LED matrix display that needs refreshing at 100 Hz.
You don't want to draw one frame and then sleep for approximately 10ms, you want to wake up at 10ms intervals after the first time you ran.

The ticks abstraction is somewhat misleading.
CHERIoT RTOS has had a tickless scheduler for a while now: there's no periodic timer interrupt, the scheduler sets up a timer interrupt for the next time that a runnable thread will wake.
Internally, the scheduler translates ticks into a wake time against the monotonic clock and suspends the thread with that position on the wake queue.
Adding absolute timeouts was a small change in the scheduler: just bypass that computation.
The hard part was maintaining backwards compatibility.

We wanted to allow blocking operations to continue to just forward the parameters that they were passed.
Anything that already took a `Timeout *` as an argument and just forwarded it should be able to keep using the parameter and passing it down to the underlying APIs.
At the same time, we wanted to avoid doubling a load of APIs.

We introduced a new type: `TimeoutArgument` to make this work.
This is a union of a `Timeout *` and an `AbsoluteMonotonicTimeout`.
Absolute timeouts are defined in cycles against the target's monotonic clock source and are represented as a 64-bit integer (note: a 64-bit integer gives a range of thousands of years at typical monotonic-counter tick rates, so we may repurpose some of the top bits in the future as a discriminator).
Because absolute timeouts in ticks are integers and the existing timeouts are pointers, we can use the *CHERI tag bit* to differentiate them.

Neither standard C nor C++ allow you to pass a `Timeout *` as an argument to a function that takes a union.
Fortunately, this is an old problem.
POSIX changed the interface of `wait` relative to what 4.1BSD shipped and so GCC provided (and clang adopted) a *transparent union* attribute, which tells the compiler to permit implicit conversions from any of the union members.
C++ did not need this attribute because you can explicitly define a constructor for a union from the members.

The monotonic API also lets us more efficiently represent the two most common values of a timeout: 0 and infinity.
A lot of callers of potentially blocking APIs want to either permit an unlimited amount of blocking or want to completely prohibit blocking.
The new APIs just require passing -1 or 0, both of which can be materialised efficiently and without needing a stack allocation.

If you're writing code that takes a timeout, please update your argument type from `Timeout *` to `TimeoutArgument`.
Existing callers will keep working, new callers will be able to provide either kind of timeout.

At the same time, we've added some new helper APIs in [`timeout.h`](https://rtos.api-docs.cheriot.org/timeout_8h.html):

 - `timeout_is_valid` checks if a `TimeoutArgument` is either an absolute timeout or a valid pointer to a `Timeout` structure.
 - `timeout_has_expired` checks if a `TimeoutArgument` has already expired.
 - `timeout_may_block` is a convenient helper that negates `timeout_has_expired`.
 - `timeout_elapse` and `timeout_elapse_from` are intended for the use case where you want to block for less than the allowed time of a timeout (e.g. one tick).
   They handle updating the timeout to indicate either a specified number of ticks (or a number of ticks from another `Timeout *`) have elapsed.

There are also some useful symbolic constants: `TimeoutWaitForever` and `TimeoutNoWait` to indicate infinite blocking or no blocking are permitted.

# It's three o'clock and all's well!

The timeouts are all relative to the *monotonic clock*.
This is the clock that the scheduler uses.
That used to be a fairly informal concept in CHERIoT RTOS because all of our simulation platforms and FPGAs had a cycle timer that elapsed at a fixed rate.
Real chips may be able to run at different speeds to trade power consumption against performance.

We've made this abstraction explicit.
Platforms are now required to implement the `platform_monotonic_time_read` function in `platform-time.h`.
This is a C API that defines how *any compartment* may read the same clock that the scheduler uses.
The scheduler interface remains in the `TimerCore` type in `platform-timer.hh`.

The read interface may be a simple CSR read or it may be exposed via a read-only view of a subset of the interface that the scheduler sees.
The latter is nice and easy in CHERIoT: you can expose a read-write view of an MMIO range to the scheduler and a read-only view of a 64-bit field in the middle to everyone else.

For a lot of embedded systems, that's the only time that matters: time intervals matter more than any absolute notion of time.
For connected systems, or anything presenting a user interface, may also need to talk about some notion of time connected to a calendar.
This has various names.
POSIX refers to it as 'real time', which is confusing in the context of real-time systems and also implies that the other times are somehow not real.
It's also commonly called the 'wall clock time', which is the name that we've used, since it's the time you might see on a clock on a wall.
Note for younger readers: before everyone had a brain implant with a chronometer, clocks were often mounted on walls so that multiple people could see the same clock and have a shared understanding of the current time.

We already had some infrastructure for getting the wall-clock time:
The SNTP compartment in the network stack exposed a shared object that allowed a `gettimeofday` implementation to read the time.
Some concept of wall-clock time was important in the network stack because TLS requires a wall-clock time to determine whether a certificate is valid.

That worked nicely as long as NTP was the *only* source of wall-clock time that you cared about, but this isn't the case for more complex systems.
For example, you might have a battery-backed clock that can be used on first boot, which you synchronise with NTP once the network is available.

The approach for addressing this is covered in [this design document](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/main/docs/Clocks.md).
We now have a compartment in the RTOS that can be extended with other components that register read-only or read-write interfaces to clock sources.
When invoked, it will query available clock sources and write the best one back to any that support storing the time.
It will also expose the current delta between the wall clock and monotonic clock via a pre-shared object so that library functions can get the current time without a cross-compartment call.

Along with this, we've also added a useful subset of the POSIX [`time.h`](https://rtos.api-docs.cheriot.org/time_8h.html) functions.
These include `gmtime` for converting from a POSIX timestamp to a human-readable Gregorian calendar version and the BSD extension `timegm` for converting back.
These do not support time zones, but limited time-zone support is easy to add on top.

# Booking it in

All of these new features need new documentation!

Work on the second edition of the CHERIoT Programmers' Guide is underway, including [a new chapter on time](https://cheriot.org/book/time.html).

The current working drafts contain the [changes from the first edition](https://cheriot.org/book/changes.html), so you can see what else has been updated.
This is all work in progress and has not yet been edited, but will eventually end up in the second edition after copyediting and review.
