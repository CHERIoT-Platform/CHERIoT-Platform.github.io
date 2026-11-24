---
layout: post
title:  "Identifiers are not capabilities"
date:   2025-01-19
categories: cheri philosophy
author: David Chisnall
---

A lot of the security of CHERI and the higher-level abstractions that we have built in the CHERIoT platform come from one very simple idea: deconflating identifiers and capabilities.
This conflation is so common and so entrenched in other systems that it will take a little while to explain what that means.

In a lot of mythology, names have power.
If you know the true name of a thing or person you can control it.
Even if you don't believe in magic, this property is true for a lot of constructed systems.

Think about, for example, the telephone system.
Have you ever made a typo in a phone number and dialled a wrong person?
The phone network still connects.
There are two problems with this, from a security perspective.
First, you are not authorised to call the person that you connected to.
Second, you are not connected to the person that you think you are connected to.

This happens because a phone number is simply an *identifier*.
Each phone has a unique number that identifies it.
If someone discovers your phone number, perhaps because you gave it to a company and they sold it to marketers, or because someone you know had their address book compromised, then they can call you.

A capability is a token that grants authority and allows you to perform an action if you present the capability.
The key for your front door is a capability: anyone who holds it can enter your house.
Your address is an identifier: it is just a label that conveys no authority.
You wouldn't want to build a system where knowing your address is sufficient to be able to enter your house.
Yet this is exactly how the telephone network works: Someone can make your phone ring just by knowing the number.

The space of phone numbers is small.
In the UK, mobile phone numbers were assigned to operators in contiguous ranges.
I've moved my phone number between operators a few times but I still get a lot of scam calls claiming to be from the operator that the number was originally assigned to.
The scammers are war dialling: calling every number in that range (a term named after [the cult film that used the technique](https://www.imdb.com/title/tt0086567/)).
It's easy to enumerate all of the phone numbers in a range and try to attack them all.
This is illegal in some places, but it's the least illegal thing scammers are doing so doesn't work as a deterrent.

This same conflation exists in email and leads to a big problem with spam.
It's picked up by Signal and other instant messaging platforms, with the same problems.

At the lowest level, non-CHERI computers have the same conflation when it comes to memory.
Simply knowing the address of a piece of memory lets you access it.
If you do some invalid arithmetic and end up with the address of a different object, you can still access it.
*This is exactly the same problem as dialing the wrong number in a phone system* and has the same root cause.
You think you are accessing one object, but are actually accessing the wrong one (bad for you), and if you modify that object then you've broken some other bit of the system (bad for the other component).

Similarly, an attacker can just try addresses until they find the correct one.
Speculative execution attacks make it very quick to probe an address space.
Simply learning the address of an object is sufficient to be able to forge something that lets you access it, because addresses identifiers are treated as capabilities.

This is very different in a CHERI system.
Knowing the address of a piece of memory does not grant you access to that memory.
You must also hold a valid capability that *authorises access to that region*.
If you hold a pointer (represented as a CHERI capability) and do some arithmetic that makes its address point to a different object, you will trap when you access it.
If an attacker guesses the address of an object, that's fine: they still don't hold a capability that authorises access to it.

In CHERIoT, this model is used at all levels of the system.
On a conventional UNIX-like system, everything that a process does to interact with the outside world happens via *file descriptors*.
File descriptors start at 0 and are numbered sequentially.
The first three are standard input, output, and error.

If a part of your program wants to access a file or a network socket, it must know the file descriptor number.
That number is just an integer that indexes into a table.
From the kernel's perspective, the thing in the table is a capability (it authorises access to a specific resource), but within your program file descriptor numbers are just integers.
If a file descriptor is closed and a new one opened, the number is reused.
Anything that holds the old file descriptor number will now use the wrong file, socket, or whatever ends up in that entry in the file-descriptor table.

In a CHERIoT system, anything like a file descriptor (for example, an endpoint for a message queue or a network socket) is represented as a *sealed capability*.
This is a tamper-proof pointer that can be handed around just like any other pointer in your C code, but can be unsealed and used only by a compartment that holds the correct rights.
If part of a program has a capability to a network socket, no other part of the program can forge a capability that allows access to that socket.
The only way to confuse two sockets is if you put the pointers to them in some data structure and index them incorrectly.
After a socket is closed, all capabilities to it are implicitly invalidated using the same mechanism that prevents use-after-free vulnerabilities on CHERIoT and nothing else can access them.

This conceptually simple change has *enormous* security benefits.
It doesn't prevent all vulnerabilities but it does eliminate a lot of them.
For example, two years ago there was a high-profile vulnerability in Microsoft Exchange where you could trick Exchange into writing over important system files, allowing an attacker to elevate privilege.
This vulnerability was possible because Exchange had the ability to write over a lot of the filesystem with elevated privileges for various reasons and it was accidentally exercising this right.

This vulnerability had nothing to do with memory safety and so CHERI alone wouldn't have prevented it, but the mindset and approach behind CHERI systems would have done.
Rather than having a handle to specific directories that allowed it to exercise privilege (capabilities) it used paths (identifiers) to write to the filesystem and implicitly used its *ambient authority* to write with more privilege than it should have used for that specific operation.
This vulnerability would not have been possible on a system that treated identifiers and capabilities as distinct.

This deconflation provides CHERIoT with spatial and temporal memory safety but that's only the start.
It is a building block for creating systems where security is the default and where the easy and obvious way of writing software leads to secure and reliable systems.
