---
layout: post
title:  "CHERIoT and the supply chain"
date:   2024-04-04T12:00+00:00
categories: rtos supply-chain auditing
author: "David Chisnall"
---

Late last week we learned that much of the world narrowly avoided a backdoor in SSH, introduced via the dependency on liblzma via libsystemd.
A malicious actor introduced a backdoor into liblzma, which could exploit SSH via this dependency chain (introduced by Linux distributions, not in the upstream OpenSSH).
This specific attack is not relevant to CHERIoT because it targets programs that have much bigger system requirements than the kinds of devices that we support, but the underlying concept is directly relevant.
This is one of the categories of attack that CHERIoT was designed to protect against.

CHERIoT provides tools at various layers to reduce the risk of this kind of attack.
Enabling fearless code reuse is one of our principle goals.
This doesn't just mean mitigating accidental bugs from memory safety errors in C/C++, it means ensuring that even an active attacker in the supply chain can do only a limited amount of damage.
In this post we'll discuss the various layers that build up to providing these guarantees and how we got to this point.

## The origins of CHERI

As a compression library, the interface of something like liblzma is quite simple.
You provide it with buffers for compressed and uncompressed data, it reads from one and writes to the other.
Ideally, the library would be completely isolated and so it would be able to read the input data, write the output, and nothing else.

This observation was made in [Capsicum: practical capabilities for UNIX](https://papers.freebsd.org/2010/rwatson-capsicum.files/rwatson-capsicum-paper.pdf) with regards to zlib (another compression library).
The paper noted that, with Capsicum on conventional hardware, it was easy to sandbox the `gzip` program that wrapped zlib, but not easy to sandbox the library.
You may notice two things about that paper:

 - One of the authors was a PI on the original CHERI research grant, another was on the advisory board.
 - It was published the same year that CHERI development started.

Neither of these is a coincidence.
CHERI was originally created to address the library-compartmentalisation problem.
This problem comes from a very simple observation:

**Isolation is easy, sharing is hard.**

If you want to fully isolate two workloads, you can run them on different computers.
The problems come when you want to share resources.
When you start looking at libraries, the problem is even more pronounced because the things that you want to share are at the granularity of objects, not files or pages.

CHERI was designed to allow you to load a library with no rights and pass it pointers to objects, which would allow it to access those objects and nothing else.
I joined the project in 2012 to lead the compilers / language strand of the research.
Our first approach involved annotating pointers to objects that may be shared so that they could use CHERI capabilities and everything else could use integer addresses to represent pointers.
It rapidly became apparent that it was not a good strategy because it meant that all code that touched an object had to be aware that the object might be shared.
Instead, we moved to the 'pure-capability' model, where every pointer was represented as a capability.
This involved orders of magnitude less porting effort and had the nice benefit of giving fine-grained memory safety for all objects.

It's worth remembering at this point that memory safety was never the goal of CHERI, it's just something that you need to be able to solve the real problems.
With object-granularity memory safety, you can build a linkage model that lets you link something like libz or liblzma and know that, at worst, it can tamper with the output buffers that you give it.
Memory safety that can't be bypassed even by assembly code in the untrusted library is a necessary building block for this kind of abstraction.

On a CHERI system, every pointer in the source language is lowered to a *CHERI capability*, an unforgeable hardware type that authorises access to memory and carries bounds and permissions.
You can think of a CHERI capability as a hardware-enforced pointer.
Every memory access (load, store, or jump) instruction takes a capability that authorises the operation as an operand.
You can shrink the bounds or remove permissions to create new pointers with restricted rights, and that's a simple register-to-register operation.
Even an attacker with arbitrary-code execution (including a supply-chain attacker who ships a malicious component) must respect these constraints.
If you don't hand a piece of code a capability to some memory, it cannot touch that memory.
In contrast, on non-CHERI systems, pointers in the source language are lowered to integer values that represent addresses.
Loads, stores, and jumps on non-CHERI systems take an integer operand and will access whatever memory is present at that address.

The CHERIoT project builds atop all of this prior work.
We wanted to build a compartmentalisation model (and an associated programmer model) for embedded systems that would allow us to address these supply-chain security concerns and fearlessly use third-party code.
CHERI was (and remains) the only viable foundational technology for this kind of system.

Although the rest of this post is specific to CHERIoT, many similar ideas can be applied on bigger CHERI systems such as [Arm's Morello](https://www.morello-project.org).
The current CheriBSD release includes [initial support for library compartmentalisation](https://ctsrd-cheri.github.io/cheribsd-getting-started/features/c18n.html) and this is rapidly evolving.
CHERIoT showcases what's possible if you can assume CHERI everywhere, in environments where recompilation is acceptable.
In contrast, CheriBSD demonstrates how easy it is to incrementally adopt CHERI in environments with strong backwards compatibility requirements.

## The CHERIoT ABI introduces boundaries

UNIX originally did not have any notion of shared libraries.
When they were added, they replicated the model of static linkage as closely as possible.
Unlike MULTICS, where libraries could be isolated domains, shared libraries in a UNIX process all run in the same address space, with the same privileges.
The liblzma attack took advantage of this to redirect symbols from the main SSH executable into its own functions.

This kind of attack is not possible on the CHERIoT platform.
The loader runs at startup (just as the run-time linker does on UNIX) but is protected.
Once untrusted code (i.e. anything other than the loader) is invoked, the compartment boundaries are in place and so one compartment cannot tamper with another compartment's imports (or any of the rest of its state).

CHERIoT provides two abstractions for code reuse; shared libraries and compartments.
Compartments are inspired by MULTICS libraries.
They have private code and globals and each introduces a new security context.
When you call a function that is exposed by a compartment, calling it involves a transition that prevents the callee and caller from seeing each other's data.
Only things explicitly passed as arguments or return values (or things reachable from them) are shared.

Shared libraries are similar to UNIX shared libraries in that they do not introduce a hard security boundary.
Functions exposed from shared libraries run in the context of their caller.
Shared libraries may contain code and read-only globals, but not mutable globals.
Because they are immutable, this is largely equivalent of copying the code into the library into each compartment that calls it.

Arguments passed at compartment boundaries can be further restricted.
For example, in our network stack, the TLS compartment (BearSSL) asks the TCP/IP compartment to provide new plaintext data from a socket.
The TLS implementation reuses its own internal buffers for partial messages.
Before passing a pointer to one of these to the TCP/IP layer, it removes all permissions except store and truncates the bounds to the available space.
This means that, even if the TCP/IP stack is completely compromised, it cannot:

 - Hold a pointer to the buffer for longer than a call (enforced with the [local / global mechanism](https://cheriot.org/book/top-concepts-top.html#permissions)).
 - Overwrite anything else in the ring buffer (which may introduce time-of-check-to-time-of-use bugs if the TLS stack reads some of a message twice).
 - Read any stale data in the ring (which could leak decrypted data).

The network stack is designed on the assumption that an attacker will gain arbitrary-code execution abilities in the TCP/IP compartment.
This wasn't done especially to protect against supply chain attacks, it's just that this compartment is the one directly exposed to network adversaries and so is the most likely to be compromised.
That compartment contains a small amount of our code and a much larger amount of code from the FreeRTOS+TCP project.
A supply chain attack on them would allow an attacker to start with arbitrary-code execution in that compartment.

By having compartmentalisation, we can reason about what an attacker can do in such an event.
Without compartmentalisation, we must assume that an attacker who has arbitrary code execution can do anything.

## Compartments can export isolated state

A lot of compartments have state associated with callers.
For example, the socket API hands back sockets to the caller, the TLS compartment hands back TLS connection state.
CHERI provides a mechanism for this, which CHERIoT uses extensively, in the form of [*sealing*](https://cheriot.org/book/top-concepts-top.html#sealing_intro).
You can think of sealed CHERI capabilities as hardware-enforced opaque types.
In C, you can return a `void*` or a pointer to a forward-declared `struct` from an API to signal that the caller shouldn't do anything with that pointer except hand it back.
This relies on trust: a caller can cast that pointer to some other type and dereference it.
A sealed CHERI capability, in contrast, can't be used or modified, except via a `cunseal` instruction.

The unsealing instruction requires a capability that matches the one used to seal the capability.
If they don't match (either because the unsealing capability is invalid, or because the opaque pointer is not of the correct type) then the operation returns an invalid capability.
You can think of this as a hardware-enforced mechanism for providing type-safe opaque types that can be shared with untrusted code.

This mechanism allows, for example, our TCP/IP compartment to hand out sealed capabilities to structures encapsulating socket state.
This sealed capability can be passed between compartments.
You can build systems where one compartment creates network connections and hands the (sealed) pointer to a second compartment.
The second compartment can then send and receive using the socket but may lack the rights to create sockets.
No compartment can forge pointers to sockets and no compartment can use a socket that it has not been explicitly passed.
When you pass any pointer to the send or receive function, the TCP/IP stack has a hardware-backed guarantee that this really is a valid socket object.

This mechanism is vital for building usable compartmentalised interfaces because it allows sharing rich types with untrusted entities.
Conventional operating systems typically provide an abstraction like file descriptors or handles to expose opaque types to userspace.
These can be opaque integers that are looked up in some in-kernel structure.
This works because there is a single kernel and it can have a separate table per userspace process.
CHERI's sealing mechanism makes this work between multiple compartments with mutual distrust.

For example, consider the TLS and TCP/IP compartments (as described in [our post about the network stack](/rtos/networking/auditing/2024/03/08/cheriot-network-stack.html))
The TLS compartment knows that any data coming over the network may be malicious and so assumes that the TCP/IP stack may have been compromised.
At the same time, the TCP/IP stack wants to ensure that the holder of one socket cannot send data over another (and may be maliciously trying to do so).
Neither compartment trusts the other.
The TLS stack also wants to be able to guarantee that the owner of a TLS session can ask it to send data, but the caller can't send unencrypted data over the same socket.
The sealing mechanism on top of a compartmentalisation model enables all of these use cases.

All state associated with a TLS session is in an object that, between calls to the TLS connection, is reachable *only* by the sealed capability.
This means that a compromise in the TLS compartment cannot affect other TLS sessions without a lot of work by the attacker (or a supply-chain attack on BearSSL).

On CHERIoT, we also have a notion of static sealed objects.
These are like global variables but where the owning compartment does not have access to the object, only to a sealed capability (opaque pointer) to them.
These are used to bootstrap various things.
For example, if you want to allocate memory then you must present an allocator capability to the allocator compartment.
This is a sealed pointer to an object containing the quota that determines the maximum amount that you can allocate.
Similarly, to create a network connection you must present the network API compartment with a sealed capability authorising you to connect to a particular host.
This means that, for example, a supply-chain attack on BearSSL would not be able to create sockets and send a copy of all plaintext to a third party, though it would be able to substitute weak cyphers for strong ones.

The core of the RTOS uses sealing to [remove the scheduler from the trusted computing base (TCB) for confidentiality and integrity](https://cheriot.org/book/top-core_rtos-top.html#_time_slicing_with_the_scheduler).
On context switch, the switcher spills registers and then invokes the scheduler with a sealed capability to the register-save area.
The scheduler then hands back a sealed capability to a register-save area.
The scheduler cannot see the state for the interrupted thread and the switcher can detect whether the capability that it's given for a new save area is a valid one.

The cross-compartment call mechanism in CHERIoT is also built on top of sealing.
Sealing is one of the most important parts of a CHERI system because it's the thing that enables all of these compartmentalisation-related abstractions.

## Compartment boundaries become auditing boundaries

Once you have defined security boundaries around third-party code, you can use them for auditing.
The CHERIoT ABI and compartment model were designed with this in mind.
A trivial compartment has just two capabilities, one to its code and one to its data.
Such a compartment is not useful: it cannot ever run.
A useful compartment needs to export at least one entry point and typically import entry points from other compartments.

To be able to invoke any entry point in another compartment, the loader must give the compartment a sealed capability for that entry point.
For library calls, this is also true, though with a special kind of sealed capability that the hardware unsealed on a jump (a sealed entry, or 'sentry' capability).

The loader will provision capabilities to compartments based on metadata filled in by the linker.
This means that the linker knows (among other things), for every compartment:

 - What entry points it exposes.
 - Which of those entry points run with interrupts enabled or disabled.
 - Which entry points in other compartments (or libraries) it calls.
 - What static sealed objects does it have (including the contents of those objects and which compartment owns the capability that authorises unsealing them).
 - What the exact contents of the compartment's code and initial values of its globals.
 - Which memory-mapped I/O (MMIO) regions it can access.

If the linker doesn't know that a compartments wants to do something outside of that compartment's address space then it can't tell the loader to provide the capabilities that authorise that action.
If the loader doesn't provide the capabilities to access something, the compartment cannot access that thing.
Like all secure designs, CHERI systems fail closed.

When you link a CHERIoT firmware image, you also get a JSON report containing all of this information.
The [`cheriot-audit`](https://github.com/CHERIoT-Platform/cheriot-audit) tool consumes this JSON and checks it against a policy that the firmware integrator writes to describe compartment structure.
For the network stack, [the included `cheriot-audit` policy](https://github.com/CHERIoT-Platform/network-stack/blob/main/network_stack.rego) checks a few things itself:

 - Is the firewall compartment the only thing that can talk directly to the network device?
 - Are the internal APIs between the various compartments in the network stack called only by the compartments that are supposed to call them?
 - Are the threads for the network stack and the firewall set up correctly?
 - Are all static sealed objects that authorise connections valid?

On top of this, the network stack's policy provides functions and rules for firmware integrators to use to check which compartments may initiate network connections, and to create allow lists of the hosts, ports, and protocols that compartments may use.
The core RTOS lets you write policies that can describe which functions may run with interrupts disabled, which compartments may allocate memory (and how much), and so on.

The goal for all of this is to allow firmware integrators to pick up third-party components and use them fearlessly.
You can define the rights any compartment has and reason about the damage that it can do if fully compromised (either by a supply-chain attack or by a later compromise arising from a bug).
You can do this at a granularity right down to what functions they may call in other compartments (and what other compartments may call them) and what objects that hold that enable this.
Supply-chain attacks aren't impossible on a CHERIoT system but they're far harder than on a conventional architecture and OS.

People often talk about CHERI as a memory safety technology but that's missing the point.
Yes, it's nice that roughly 70% of security vulnerabilities are mitigated by a CHERI system (with a temporal safety solution such as [Cornucopia Reloaded](https://www.cl.cam.ac.uk/research/security/ctsrd/pdfs/202404asplos-cornucopia-reloaded.pdf) or CHERIoT) but that's just the start.
The real power of CHERI comes from being able to build compartmentalised software stacks where supply chain attacks (and other compromises) can be mitigated and where the you can fearlessly reuse third-party code.
