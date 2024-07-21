---
layout: post
title:  The CHERIoT philosophy is designed to prevent things like the CrowdStrike disaster
date:   2024-07-19
categories: security philosophy
author: David Chisnall
redirect_from:
  - /security/philosophy/2024/07/03/crowdstrike-is-the-opposite-of-cheriot.html
---

The news today is full of reports about a Windows kernel driver from CrowdStrike causing large numbers of computers to be unbootable.
Each step that resulted in this catastrophe involved a choice that is the exact opposite of the philosophy behind CHERIoT

On a Windows system, the absolute highest privilege mode is the virtual secure mode, which is used for a fairly limited set of core system services.
Below that is kernel mode, which is almost as powerful.
A bug in kernel mode can access any file on the system, tamper with any user data, and (as today's news showed) cause the entire system to fail.
Historically, conventional monolithic OS kernels have been a mixture of an overtly security-critical core (with its need for a privileged perspective), device drivers, and even non-essential but performance-sensitive software.
There has been little internal isolation between these different aspects of the kernel.
Each new feature adds to the attack surface and the overall fragility of the system.

Secure software development starts with the principle of least privilege.
A 'secure by design' approach means that nothing should be in the kernel unless it absolutely needs to be.
The first mistake for the CrowdStrike bug was putting code in the kernel that didn't need to be there.

In CHERIoT, the switcher, which handles transitions between compartments and between threads, is the closest equivalent of a kernel.
This is a very small amount of code, [from a single file](https://github.com/microsoft/cheriot-rtos/blob/main/sdk/core/switcher/entry.S).
Everything else, including OS services such as thread scheduling, lives outside of this core and runs with only the set of privileges that it needs to fulfil its role.

It appears, from reverse engineering reports, that the problem arose from CrowdStrike parsing a corrupted rules file and dereferencing an invalid pointer.
In other words, it's the kind of memory-safety bug that CHERI was designed to protect against.
CHERI does not prevent you from doing this kind of thing (though language features can make it much harder).
But wait, you might ask, CHERI *also* traps on invalid memory accesses, how is that better?
The difference is the impact of that trap.

In a system designed for compartmentalisation, failure is isolated.
We talk a lot about the confidentiality and integrity legs of the CIA tripod, but availability is also very important.
A CHERI system has two significant benefits relative to the behaviour that we saw with CrowdStrike.

First, the trap is guaranteed to happen *before* a memory-safety error corrupts any data.
If you attempt an out-of-bounds write, you trap before anything hits memory, not after it's corrupted an unspecified amount of data and the next invalid operation happens to hit an unmapped page.
This means that recovery is generally possible.

More importantly, the failure is scoped to a compartment.
If a service on a CHERIoT system encountered this kind of error, that compartment would invoke its error handler (if it has one).
Even if it isn't able to gracefully recover, the fact that it failed would not bring down the entire system.
Only compartments that directly called into the failed compartment would be affected.

If a system repeatedly fails to boot or crashes or fails to feed a watchdog in a sufficiently tight loop that availability would be compromised, it should fall back to a minimal configuration that suffices to contact the update servers.
Compartmentalization also makes it easier to have an A/B recovery and keep at least one recovery image in a known-good state.
During the system's normal operation, after the loader has finished, the online update compartment would not be authorized to write to the most recently successfully used recovery image, ensuring that the system could recover again if needed.
During a recovery operation, only the ordinary system image would be writable, ensuring that any attempted recovery did not brick itself.

I would be willing to bet that most CISOs that mandated CrowdStrike across their organisations were completely unaware that it had the ability to completely incapacitate their companies.
This is where one final advantage of a CHERIoT system comes in.
The [auditing infrastructure](https://cheriot.org/rtos/firmware/auditing/2024/03/01/cheriot-audit.html) that we provide lets you inspect compartment interactions at link time and write policies.
If you have a management compartment from a third-party vendor and want to ensure that it cannot impact the functionality of business-critical components, you can write policies to enforce this.

The principle of least privilege means never having to say sorry I caused a worldwide IT system outage.
