---
layout: post
title:  "Why the alleged ESP32 backdoor couldn't happen here"
date:   2025-03-09
categories: auditing backdoor
author: David Chisnall
---

If you've been following the news this weekend, you'll have seen [articles about](https://www.bleepingcomputer.com/news/security/undocumented-backdoor-found-in-bluetooth-chip-used-by-a-billion-devices/) [a vulnerability (alleged to be an intentional backdoor) in ESP32 microcontrollers](https://nvd.nist.gov/vuln/detail/CVE-2025-27840).
The news is somewhat overhyped (the attacks *probably* require physical access) but it provides an opportunity to look at what we did in CHERIoT to eliminate this class of attack by construction.

# Binary-only drivers may be unavoidable

The vulnerability is in binary-only Bluetooth drivers.
The obvious response is 'use only open-source code'.
That's a nice place to be, and one that (as an open-source platform, we strongly encourage!) but it's not always possible and doesn't always fix the problem.

First, being able to read the code and build it yourself makes it *possible* to find a bug, but only if you're doing a careful code review.
Auditing code that someone else has written is notoriously hard and, as the [underhanded C contest](https://www.underhanded-c.org) showed, hiding intentional vulnerabilities in source code is much easier than finding them.
Even finding unintentional bugs is hard, as the [70 published CVEs for the Linux kernel so far this year](https://www.cvedetails.com/vulnerability-list/vendor_id-33/product_id-47/Linux-Linux-Kernel.html?page=1&year=2025&month=-1&order=1) attest.

Second, open-source code may not be an option.
Modern radios for wireless networks often have a software-defined component that enforces regulatory compliance.
The WiFi standards, for example, all define a set of bands that is the union of all frequency ranges that *any* regulatory regime permits.
To ship a device in a particular country, you may be required to lock down the set of bands that it will use to the ones that are allowed in that country.
Depending on who is responsible for the certification, that may mean that a device vendor is required to run a specific version of a driver, provided by a component vendor.

# Your driver can access what?

The vulnerability contained a set of undocumented commands that could be sent to the device that included arbitrary memory and flash reads and writes.
If you expose the Bluetooth interface to untrusted code (for example, allowing commands to be sent to it over the UART) then that code has total control over the device, including the ability to rewrite the firmware on flash and install persistent malware.

Devices configured in this way are probably a lot less common than the news would suggest, but let's imagine a slightly stronger version of this vulnerability that took those commands from the network side.
What would that mean on CHERIoT?

First, nothing in a CHERIoT system that runs after the system has finished booting has access to *all* memory.
An arbitrary 'read memory' or 'write memory' primitive simply cannot exist.
At worst, a driver could leak or corrupt all data that the compartment owning the driver has write access to.
This does not, for example, include the compartment's code (which is read-execute, not writeable).

# Auditing makes it impossible to hide this kind of backdoor

So precisely what *can* a driver access?
That's what [CHERIoT's auditing flow](https://cheriot.org/book/audit.html) is designed to tell you.
When you build a CHERIoT firmware image, the linker emits a report explaining everything that every compartment has access to.

In the CHERIoT network stack, the network device is owned by the firewall compartment, which instantiates the network device driver provided by the specific target platform.
This compartment is trusted for network availability (it can always drop incoming or outgoing network packets) and is trusted for confidentiality and integrity of any unencrypted network traffic.

If you look at the audit report for this compartment you can see that it has access to:

 - A bunch of C library routines (`memcpy`, atomic add, and so on friends).
 - The MMIO region for the network device.
 - A capability that authorises it to wait on and acknowledge the interrupt associated with the network device.
 - Allocator APIs for memory allocation and a capability for a small heap quota.
 - Scheduler APIs for waiting on a futex.
 - An API in the DNS compartment to forward DNS replies (and another that initialises the DNS compartment).
 - An API in the TCP/IP compartment to forward incoming packets that are not DNS responses.
 
This is an exhaustive list.
Even if the network driver (which is compiled into the firewall compartment) is malicious, this lets you reason about everything that it can do.
It cannot expose an API to write all memory, because it does not have access to all memory.
It cannot expose an API to write to flash, because it does not have access to all flash.

These guarantees hold no matter how an attacker gains arbitrary-code execution in that compartment, whether by exploiting an unintentional bug or a back door maliciously introduced in the supply chain.

# How much further does CHERIoT go?

We designed the CHERIoT software model to simultaneously support:

 - Regulatory requirements that a particular device must be accessible to only a specific binary driver.
 - Proposed right-to-repair legislation requiring users to be able to modify all of the other behaviour on the device.
 - Strong, auditable, security properties between components provided by different people.

You can build a code signing flow with our auditing tools that signs any firmware as long as the device is accessible only to code with an exact binary hash (which you may generate from SBOM tooling) and, at the same time, add policy checks that limit what that component can access.
Even if you have to run some binary-only driver, you can ensure that it respects the principle of least privilege and has no access to anything except the device and other things that you explicitly pass to it.

This is what is possible when you design a platform for *usable* security from the ground up, rather than trying to retrofit it to abstractions for scaled-down models of 1970s minicomputers.
