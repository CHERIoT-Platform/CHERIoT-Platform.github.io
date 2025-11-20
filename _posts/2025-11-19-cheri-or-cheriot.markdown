---
layout: post
title:  "CHERI or CHERIoT?"
date:   2025-11-19
categories: cheri philosophy isa
author: "David Chisnall"
---

Recently, a few people have asked me 'should we do CHERI or CHERIoT?'
I hadn't previously written an answer to this because the question doesn't make sense: it's like asking 'should we do MMUs or x86 page tables?'
Since it's been asked several times, I think it's worth taking some time to explain *why* the question doesn't make sense.

CHERI is an abstract architecture
---------------------------------

CHERI is a conceptual extension to conventional computing architectures define a *capability model* for accessing memory within an address space.
There are a *lot* of concrete instantiations of this conceptual model.
The [original CHERI research was done on a 64-bit MIPS variant.](https://www.cl.cam.ac.uk/research/security/ctsrd/pdfs/201406-isca2014-cheri.pdf)
[Arm Morello extended ARMv8 with CHERI extensions](https://www.arm.com/architecture/cpu/morello).
The University of Cambridge [CHERI ISA v9 Technical Report](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-987.pdf) contains a RISC-V CHERI adaptations and even a sketch of CHERI for x86.
And there is an ongoing effort to standardise [RISC-V CHERI base architectures](https://github.com/riscv/riscv-cheri).

These all provide the same core set of features.
Most software isn't written for a single target instruction-set architecture (ISA), it's written assuming a set of language-level abstractions that can be represented in a particular target.
Software written for CHERI in C (or any language that is higher-level than C) doesn't care the bit patterns of capabilities, how bounds or permissions are encoded, and so on.
It cares only that you have some capability registers, that you can store capabilities in memory, and that the C language notion of a pointer can be represented by a CHERI capability for the target ISA.

I like to use memory-management units (MMUs) as an analogy for CHERI.
MIPS and SPARC had software-managed translation-lookaside buffers (TLBs).
x86 and Arm had architectural page tables that hardware walked to fill the TLB automatically on miss.
PowerPC had an MMU design that, in the words of Linus Torvalds, 'can be used to frighten small children'.
All of these implemented the same set of abstractions in terms of processes, shared memory, copy-on-write regions, and so on.
All of these are MMU designs, just as there are many possible CHERI implementations.

There are lots of things like this in ISAs.
Arm's Neon, PowerPC's AltiVec, and x86 SSE are all SIMD extensions, which provide similar programming models, but they all have quite different low-level details.
It's quite possible to write code that targets all of them.

CHERIoT is a concrete platform
------------------------------

CHERIoT is a complete ISA specification, co-designed with a software stack.
In the same way that x86 page tables or MIPS' software-managed TLB are both concrete instances of the abstract idea of a memory-management unit, CHERIoT is a concrete instantiation of CHERI.

This means that there is no 'CHERI or CHERIoT' choice.
The real choices are between CHERI or not CHERI, or between CHERIoT and some other concrete instantiation of CHERI.
Having worked on CHERI since 2012, I am obviously biased on the first of these: given a choice between CHERI and not-CHERI, you should definitely pick CHERI!
The second choice (CHERIoT or some other CHERI variant) is more nuanced, as I'll discuss more in the rest of this post.
There are good reasons for CHERIoT but there are also domains where it is not the best choice.

CHERIoT is a *rich* CHERI platform
----------------------------------

A minimal CHERI specification provides pointer integrity, bounds enforcement, and some permissions.
Pointer integrity means (among other things) that you can precisely differentiate between pointers and other kinds of non-pointer data in memory and so can build temporal safety on top.
Most of the CHERI temporal safety work has provided users with protection against use-after-reallocation issues.
A pointer in a CHERI system may point to deallocated memory, but it will not point to *reallocated* memory.
Something in the system will ensure that deallocated pointers are gone before the memory allocator reuses memory.
This means that any use-after-free bug will either point to the old object, or will trap.
It will never point to a new object and cause accidental aliasing.

CHERIoT builds on top of the core CHERI ideas with a couple of hardware features that guarantee deterministic trapping for use-after-free bugs.
CHERIoT also provides a [rich sealing design that works with our software model](/rtos/sealing/2025/11/06/sealing.html).
It extends the sentry mechanism in CHERI systems [to provide stronger control-flow integrity properties](/isa/ibex/2024/06/26/sentries-cfi.html) than most other CHERI platforms, and also uses them to enforce interrupt control.

The entire system is designed to allow [fine-grained auditing of compartment rights](/rtos/firmware/auditing/2024/03/01/cheriot-audit.html), which lets us respect the principle of least privilege by having compartments as small as a single function *and statically reasoning about what they can do in the presence of a compromise*.
This set of building blocks lets us do things like [privilege separate the network stack and make callers respect the principle of intentional use, again with auditable guarantees that let you reason about which compartments can talk to which remote servers](/rtos/networking/auditing/2024/03/08/cheriot-network-stack.html).

This adds up to an easy-to-use programmer model and richer security guarantees than, we believe, any other system (CHERI or otherwise).

CHERIoT is designed for resource-constrained systems
----------------------------------------------------

Some of the strengths of CHERIoT come from the fact that we were willing to tailor the system to small devices.
Prior CHERI systems rely on the MMU to support concurrent revocation as their mechanism for temporal memory safety.
CHERIoT does not include or need an MMU, it uses an alternative approach for temporal safety, inspired by our previous CHERI+MTE work

You don't want an MMU on an embedded device.
An MMU is typically larger than an entire microcontroller core!
Page tables are large data structures and they enforce a minimum granule size for memory protection and accounting.
A minimal process isolated with a typical MMU needs one page for the stack, one page of code, and one for read-write globals.
On top of that, it needs a complete page table, which is at least two pages.
This means you need a minimum of 20 KiB for an isolated component, of which 8 KiB is purely book-keeping overhead.
On a CHERIoT system, a lot of our compartments are smaller than the page tables that would be necessary to isolate them.
In addition, MMUs bring nondeterminism, which is not ideal in any system with even fairly soft realtime requirements.

The temporal memory system in CHERIoT is designed to scale across the range of microcontroller cores, including low-core-count multicore systems.
We check whether capabilities are valid when you load them.
Most pointers are used more than once, so this is more efficient than checking on every use (as an MTE system must do), but does not scale as well to large multicore systems or complex memory hierarchies.
You would not want to build a temporal-safety system like ours for a large server system, for example.

CHERIoT also extends CHERI's sealed function pointer ("sentry") mechanism for interrupt control.
This has some enormous benefits for the programmer model.
It is quite easy to implement on microcontroller-class systems, even dual-issue designs such as CHERIoT-Kudu.
It would be very hard to scale to big out-of-order cores.

And that's all fine, for the same reason it's fine to run different operating systems on different scales of core: different abstractions and different optimisations make sense at different scales.
By the time you've built a system where you start to run into scalability challenges with CHERIoT, you have enough area that an MMU doesn't add much overhead and enough RAM that you would benefit from running an OS designed for larger computers.

What about standard RISC-V CHERI?
---------------------------------

The RISC-V standardisation process has an unenviable task of trying to create a standard base for RISC-V CHERI systems that will work for 2-stage microcontrollers, 25-stage superscalar out-of-order server chips, GPUs, AI accelerators, SmartNICs, and so on.
This is likely to end up being a (hopefully small) family of bases and a set of extensions.

If all goes well, CHERIoT v2 will be one of those bases plus some extensions.
RVB26 will be a RISC-V profile assembled from those bases and a set of extensions necessary to run a CHERI Linux or CheriBSD.

As with the rest of RISC-V, any useful chip implements a base architecture and a set of extensions.
Profiles exist to corral a set of extensions that software can depend on in environments where binary compatibility is important, such as those running off-the-shelf operating systems and programs.

So when should I use CHERIoT vs some other CHERI?
-------------------------------------------------

If you are looking for a microcontroller-class system, CHERIoT is probably the right answer.
It is co-designed with a compartmentalisation model and a rich set of software abstractions (implemented in CHERIoT RTOS), and provides temporal safety as a baseline feature.
It is a mature and stable target, supported by multiple organisations.

If you are looking for an application core, CHERIoT is not for you.
There are a few cases where people have moved to using Linux on an Arm A-profile system and could run the same workload on a cheaper CHERIoT system, but don't expect to be able to run arbitrary Linux workloads on a CHERIoT system (ever).
The upcoming CHERI RISC-V profile is a far better choice for these use cases.
Capabilities Limited and lowRISC are implementing this in the open-source CVA6 core, so there will soon be a path to a useful open-source CHERI core in this part of the design space.
Codasip's X730 core is also available for commercial licenses and is targeting this spec (and pre-standard versions of it until it is ratified).
Hopefully there will be more implementations available in coming years.

If you are looking for an ISA as a base for an accelerator or coprocessor, CHERIoT *may* be the right choice, depending on the rest of the system.
Depending on your requirements, you may want to do something completely custom.
As long as it provides the same underlying security guarantees, it can still be CHERI!
