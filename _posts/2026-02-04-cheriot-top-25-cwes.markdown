---
layout: post
title:  "CHERIoT vs the top 25 CWEs"
date:   2026-02-04
categories: cwes
author: David Chisnall
---

Each year, MITRE publishes a list of the top 25 most dangerous software weaknesses.
The [2025 list](https://cwe.mitre.org/top25/archive/2025/2025_cwe_top25.html) is interesting reading.
Let's see how CHERIoT fares against them.

The top three (CWEs [79](https://cwe.mitre.org/data/definitions/79.html), [89](https://cwe.mitre.org/data/definitions/89.html), and [352](https://cwe.mitre.org/data/definitions/352.html)) are not typically applicable on embedded platforms.
Two are cross-site things that apply to web applications, one is SQL injection.

At position 4, we have [CWE-862](https://cwe.mitre.org/data/definitions/862.html), missing authorisation.
This is not something that's automatically mitigated by CHERIoT, but the design of CHERIoT RTOS and the programming model that we expose makes it easy to write code that avoids this kind of issue.
The CHERIoT pattern for any operation that you do on behalf of another compartment is to require an authorising capability.
For example, if you allocate memory, you must present an allocation capability that encapsulates your right to allocate memory (and your quota).
If you want to create a socket, you must present a capability that authorises you to bind to a specific port (for server sockets) or connect to a specific remote host.
The same applies for dynamically created things, such as sockets themselves, message-queue endpoints, and so on.
If you forget to authorise something, it will not have the capability to perform the action and the action will fail.
This is a general property of capability systems and not something specific to CHERIoT.

Position 5 is [CWE-787](https://cwe.mitre.org/data/definitions/787.html), out-of-bounds write, also known as a buffer overflow.
This one is deterministically mitigated by any CHERI platform.

Technically, CHERIoT is not vulnerable to the path-traversal bugs in position 6 ([CWE-22](https://cwe.mitre.org/data/definitions/22.html)), but only because we don't yet ship a filesystem.
But, again, this kind of issue is a solved problem in capability systems.
[Capsicum](https://www.cl.cam.ac.uk/research/security/capsicum/freebsd.html), for example, eliminates this kind of vulnerability and I'd expect our filesystem APIs to follow a similar shape.
There's no excuse for writing APIs that are vulnerable to path traversal in the 2020s.

The next two are good old-fashioned memory-safety vulnerabilities.
Position 7 is use after free ([CWE-416](https://cwe.mitre.org/data/definitions/416.html)), and 8 is out-of-bounds read ([CWE-125](https://cwe.mitre.org/data/definitions/125.html)).
The latter is mitigated by any CHERI platform.
The former is usually made unexploitable by CHERI systems, and is deterministically mitigated by CHERIoT.

The next two are unlikely to apply to embedded platforms.
[CWE-78](https://cwe.mitre.org/data/definitions/78.html) at position 9 is largely to do with failing to validate dynamically created command lines that you pass to a shell.
Then [CWE-94](https://cwe.mitre.org/data/definitions/94.html) (Improper Control of Generation of Code) at position 10 is typically introduced with scripting languages producing output that can be influenced by an attacker and is then executed by another interpreter, a rare situation on embedded devices.

Position 11 ([CWE-120](https://cwe.mitre.org/data/definitions/120.html)) is a 'Classic Buffer Overflow', i.e. something that CHERI deterministically mitigates.
Not to be confused with the buffer overflow we had at position 5.

The 12th entry is another that's rare on embedded devices.
[CWE-434](https://cwe.mitre.org/data/definitions/434.html) relates to unrestricted uploads of dangerous file types, something that matters a lot to web apps and far less to other classes of program.

Next, position 13, is a null-pointer dereference where a valid pointer was expected ([CWE-476](https://cwe.mitre.org/data/definitions/476.html)).
CHERI guarantees that this will trap (even if an attacker can provide arbitrary offsets to the null pointer) and CHERIoT makes this a recoverable error either via our [scoped error handlers](https://cheriot.org/rtos/errors/2024/09/20/error-handling.html) or by simply unwinding the compartment to the caller.

Buffer overflows seem to be popular and position 14 is the third instance of this kind to make the list, this time on the stack ([CWE-121](https://cwe.mitre.org/data/definitions/121.html)).
Again, this will deterministically trap on any CHERI platform.

The next entry is more interesting.
Unsafe deserialisation of untrusted data ([CWE-502](https://cwe.mitre.org/data/definitions/502.html)) is something a lot of people get wrong.
[Phil Day wrote about how to do this safely a couple of years ago](https://cheriot.org/security/philosophy/2024/07/30/configuration-management.html).
Lightweight compartmentalisation makes it easy to limit the scope of damage that this kind of bug can do, to almost nothing.

Did I mention that buffer overflows are a recurring theme on this list?
Position 16 ([CWE-122](https://cwe.mitre.org/data/definitions/122.html)) is yet another buffer overflow, this time on the heap.
One more that any CHERI platform deterministically mitigates.

Positions 19–21, sadly, are not mitigated by CHERI.
These all relate to incorrect access control at the application layer.
Position 24 is similar.

In between these, we have another web app problem ([CWE-918](https://cwe.mitre.org/data/definitions/918.html), server-side request forgery) and another command injection ([CWE-77](https://cwe.mitre.org/data/definitions/77.html)).
These are unlikely to be present on embedded devices.

Finally, at position 25, we have a fairly broad category of availability issues that arise from not constraining resource allocation ([CWE-770](https://cwe.mitre.org/data/definitions/770.html)).
These are normally mitigated by the software capability layer on CHERIoT.
For example, a compartment can't allocate memory unless it has a capability that authorises it to do so.
That capability encapsulates a quota and so provides a limit to the total amount of allocation.
Other resources that can be dynamically allocated are normally managed in the same way.

So, what's the final score card?

Not applicable in embedded contexts: 1, 2, 3, 9, 10, 12, 22, and 23

Deterministically mitigated with just a recompile: 5, 7, 8, 11, 14, and 16.

Mitigated by CHERIoT design patterns and software model: 4, 6, 13, 15, 25.

That still leaves six that we don't mitigate (17, 18, 19, 20, 21, and 24), but now hopefully the cognitive load is much lower from not having to think about the eleven that we do prevent and you can avoid some of these as well!
