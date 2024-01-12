---
layout: post
title:  "Porting from FreeRTOS"
date:   2024-01-12T12:20+00:00
categories: porting freertos
author: "David Chisnall"
---

Before writing CHERIoT RTOS, we evaluated whether we could adapt an existing RTOS to a CHERI platform.
Unfortunately, we found two things that made this hard.
First, most existing RTOSs began life on platforms with no possible mechanism for isolation and where every byte mattered.
This meant that they often lacked even software-engineering boundaries around components (for example, we found optional ThreadX components that directly manipulated internal data structures of the ThreadX scheduler), let alone security boundaries.
This meant that it was often impossible to port without a complete rewrite.

Second, in places where it was possible, the abstractions that CHERI enables, such as fine-grained sharing and function calls between compartments, enabled patterns that were not possible on existing hardware.
For example, FreeRTOS has a notion of a restricted task, which is (roughly) similar to a userspace process on a conventional operating system.
We can enforce that abstraction with CHERI, but it gives you a fairly limited benefit relative to an MPU.
At the same time, the sweeping changes required to make something like FreeRTOS fully take advantage of a CHERI architecture would make it incompatible with the large set of targets that it currently supports.

CHERIoT has always been a hardware-software co-design effort and CHERIoT RTOS was designed alongside our ISA, compartmentalisation model, and compiler.
At the same time, compatibility with existing code has always been a goal of CHERI work.
We don't want to force you to rewrite all of your software in a new language, or even in a new dialect of an existing language.
We want to be able to take existing components and use them with minimal (ideally zero) code changes.

Just before Christmas, we added [a FreeRTOS compatibility layer](https://github.com/microsoft/cheriot-rtos/commit/b5839ad25c09d91ef073e5d2226dcaf7b029b0e9) to CHERIoT RTOS.
This is sufficient for us to be able to run [FreeRTOS+TCP v4.0](https://www.freertos.org/FreeRTOS-Plus/FreeRTOS_Plus_TCP/index.html).
This is a TCP/IP stack, with IPv4 and v6 support, written in MISRA C.
Given that such code exists and is permissively licensed, there is a huge opportunity cost and very little benefit in rewriting it.

Instead, we can compile this code and run it in a compartment with almost no changes.
CHERIoT RTOS does not allow dynamic thread creation and so we wrap one file (`FreeRTOS_IP.c`) to replace the task creation call for the worker task with a statically created thread that waits on a futex and then starts running when the task-create call is reached.
We don't modify any of the source files and compile the full 21.5 KLoC as-is.
The only other necessary additions are drivers for the Ethernet interface on the Arty A7.
The one on my desk is happily joining my local network and making TCP connections and sending / receiving UDP datagrams, with both IPv4 and v6.

I've added a bit more to the [Porting from FreeRTOS](https://cheriot.org/book/top-porting_from_freertos-top.html) chapter of the [CHERIoT Programmers' Book](https://cheriot.org/book/) to help people understand the different abstractions and how to map from FreeRTOS to CHERIoT RTOS abstractions.
If you have a component that runs on FreeRTOS and are interested in how hard it would be to move it to CHERIoT, try it and see!
