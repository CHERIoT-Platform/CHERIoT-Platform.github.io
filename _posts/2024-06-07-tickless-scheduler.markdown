---
layout: post
title:  "Moving CHERIoT RTOS to a tickless model"
date:   2024-06-07
categories: scheduler
author: "David Chisnall"
---

The CHERIoT RTOS scheduler is a fairly traditional RTOS scheduler.
A typical desktop or server OS tries to ensure that all threads run, but that higher-priority threads get larger slices of available compute time.
It may also try to ensure that interactive tasks run with lower latency.
In contrast, an RTOS scheduler needs to give predictable performance and allow high-priority threads to monopolise the CPU.

The CHERIoT RTOS scheduler runs the thread that is in a runnable state and has the highest priority of all runnable threads.
If two threads are runnable at the same priority level, it round-robin schedules between them.
Lower-priority threads will run only if the higher-priority threads are sleeping (either as an explicit sleep, or waiting for some event such as an external interrupt or a lock).

The original scheduler design used the classical approach of registering for a timer interrupt with a fixed period.
Each time the timer fired, the scheduler would run, wake any threads sleeping with expired timeouts, and move to the next thread in the current priority list.
This causes two forms of wasted work.

First, if there is only one thread at the highest priority level, the scheduler would make a new scheduling decision on each tick, but that decision would always be to keep running the interrupted thread.
The running thread would be interrupted, the scheduler would do some calculations, and then the thread would resume.

Second, it meant that there was no good way to make the core enter a lower-power state.
If every thread is sleeping for an extended period, the scheduler would still get periodic timers.

We've now moved the scheduler to a tickless model.
The scheduler now calculates the next time that it may need to make a scheduling decision and requests a timer interrupt for that time.
If there is another runnable thread at the same priority level as the running thread, the scheduler requests a timer interrupt one tick in the future to preempt the current thread.
Otherwise, it requests a timer at the next timeout.
If every thread is blocked waiting for non-timeout events, no timer interrupt will be fired and the CPU can sit in the wait-for-interrupts state until an external interrupt fires and then resume.

This, obviously, should improve performance and so I was surprised when I first tested it and found that our test suite ran several percent slower.
Examining the root cause of this was quite interesting.
Our existing sleep API was being used for two quite unrelated purposes:

 - A thread waiting for some time to elapse.
 - A high-priority thread wanting to ensure a lower-priority thread was able to make progress.

Most uses were of the second form, yet the new tickless implementation did nothing to address this use case.
If you slept for three ticks, it would not wake you up until three ticks worth of time had elapsed.
If you just wanted to allow other threads to make progress but they only had one or two ticks worth of work to do, the core would remain idle until three times the tick duration had elapsed.
Slight changes to the accounting meant that we'd spend very slightly longer with the core idle.

After observing that, we refactored the `thread_sleep` API to properly express intent.
It now takes a second `flags` parameter that can specify either a yield or a sleep.
If you sleep, you will not be woken up until the number of ticks that you've requested have expired.
If you yield, you will be woken up either when the requested duration expires or if the system reaches a point where no other thread will become runnable in that period.

We have made yielding the default behaviour because almost all of the calls to `thread_sleep` wanted to yield to allow another thread to make progress, rather than to guarantee some time had elapsed.

This makes the test suite complete in around 30% faster on Sonata.
We don't expect this much speedup for other workloads.
The test suite spends a lot of time testing concurrent operations and has a lot of wait-for-a-worker-thread-to-do-something sleeps.
It should still hopefully improve performance for other workloads.

If anyone is interested in further improving the scheduler, there are still some to better track the time that equal-priority threads spend yielding to ensure fairness when a thread sleeps before the end of its quantum but its priority peer does not.

