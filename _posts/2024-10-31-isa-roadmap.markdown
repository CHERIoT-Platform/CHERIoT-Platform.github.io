---
layout: post
title:  "The plan for the CHERIoT ISA 1.0 and beyond"
date:   2024-10-31
categories: isa roadmap
author: "David Chisnall"
---

The CHERIoT ISA has been fairly stable for a while now.
The main addition has been the newer sentry types that allow us to differentiate between function pointers and return addresses and to better protect interrupt-disabled functions from abuse.
There have also been a few small fixes to corner cases.
You can read these in the change log in the [current live version of the specification](https://cheriot.org/cheriot-sail/cheriot-architecture.pdf) (Appendix A).

We aim to stabilise a version 1.0 and have a project [tracking open issues to be resolved before then](https://github.com/orgs/CHERIoT-Platform/projects/1).

The 1.0 release will include a stable abstract machine and something that we expect to support in the toolchain for a long time.
This will be referred to as the Xcheriotv1 extension to RISC-V.

CHERIoT does not compete with the RISC-V Zcheri standards
---------------------------------------------------------

People often ask me if they should use CHERIoT or the standard RISC-V CHERI32 extension.
The answer, of course, is 'yes'.

CHERIoT is a mature extension that various people have been running software on for several years and that has been refined based on that experience, but it's also an extension that is tailored for small devices.
We made a number of choices that let us scale down to 3-stage single-issue pipelines with tens of KiBs of RAM, which would be the wrong approach on multicore, multi-GHz superscalar out-of-order architectures with TiBs of RAM.

The CHERI standardisation effort, in contrast, has to produce something that works everywhere.
It is defining a set of extensions under the Zcheri umbrella.
It also has to produce something that is a minimum viable product, with a focus on memory safety.
A lot of things in CHERIoT have demonstrated value in our use case will not make it into a 1.0 release of the standard CHERI extensions.

The things added to RISC-V by CHERIoT can be split into several categories:

 - Things that we think everyone should do.
 - Things where we've made compromises in a direction are necessary for small devices.
 - Things where we've co-designed the hardware and software and, while we think they're the right approach in a purist sense, may not be the most compatible options.

We've been working with the RISC-V International CHERI SIG and TG on ensuring that the Zcheripurecap extension is a subset of CHERIoT.
We believe that, in the current drafts, everything in our first category is in the RVI standard and everything in the second and third categories are permitted.

This will become important for CHERIoT 2.0.

CHERIoT ISA v 2.0
-----------------

Our goal with the 2.0 release of the CHERIoT ISA is to rebase on top of the official Zcheripurecap extension, once this has been ratified.

We expect to be able to provide compatibility between 1.0 and 2.0, such that anything that isn't machine code can run on both.
We anticipate that RISC-V International will assign new opcodes for the CHERI instructions and so this release won't be binary compatible, but we expect to be able to maintain source compatibility.

This will likely require a compatibility file of assembly macros because the official standard renames most instructions to make them ~~less readable~~ more consistent with RISC-V naming conventions.
For C and higher-level languages, the compiler will produce different instructions, but with the same abstract machine.

This compatibility means that it should be trivial to support 1.0 and 2.0 for the same amount of time.

We may propose other parts of our extensions as future Zcheri standards, if they have value beyond embedded devices.

What does CHERIoT have that Zcheri doesn't?
-------------------------------------------

CHERIoT makes a lot of use of one of the concepts that's been in CHERI since the beginning: sealing.
Sealing is how we build type-safe opaque types.
This, in turn, is how we build software capability systems.
We believe sealing is essential to providing a robust compartmentalisation model, but it's not necessary for single-trust-domain memory safety.

Zcheri defines a reduced subset of sealing, which is sufficient to provide sentries, but not the rich set of features that we use.
The core ISA leaves room for expansion and so this is easy to add to the current drafts as a CHERIoT-specific extension (and possibly an optional Zcherisealing extension later).

CHERIoT has a richer set of sentries, including those for enabling and disabling interrupts on jump and capturing the prior state in the return capability.
This is very useful for providing auditable non-interference guarantees to hard realtime systems.
It's also something that would be *incredibly* painful to implement on a wide superscalar architecture because it means that disabling interrupts may happen in speculation, in a data-dependent way.
Adding this to the core Zcheri specifications would be a terrible idea.

Outside of the core Zcheripurecap specification, there is a Zcheri extension for *levels*.
This is a generalisation of the local/global mechanism that we use to implement no-capture guarantees.
In CHERIoT, we don't have a need for more than the two levels (it would make reasoning harder), but on systems with virtual memory, hypervisors, and so on it is valuable.
We believe that our local/global behaviours can be mapped to the underlying model as a simple two-level case for this.

The biggest place where CHERIoT diverges from RISC-V is in reducing the shift of the `AUIPCC` instruction (the capability version of `AUIPC`).
In RISC-V, materialising a PC-relative address is done in two stages.
First, AUIPC adds an immediate value that is left-shifted by 12 to PC, and then a subsequent instruction (either an add, or the immediate in a load or store) adds in the remaining displacement.

Unfortunately, this means that (in the worst case) the result of the `AUIPC` instruction can go 4095 bytes too far.
On normal RISC-V, that's fine: it's just an integer.
With CHERI, this may be 4095 bytes out of bounds of the program counter capability.

When you scale a capability encoding down to 32 bits and short pipelines, representing 4 KiB out-of-bounds values adds some unfortunate complexity.
We have some proposed formats that will allow this, but microarchitects have estimated that it will hit the maximum frequency of short pipelines by as much as 10%.

To accommodate encodings with a restricted out-of-bounds representable region in CHERIoT, we reduce the shift in `AUIPCC` by one.
This means that there's a one-bit overlap between the immediate in add / load / store instructions and `AUIPCC`.
The loader can determine the direction of the addition in `AUIPCC` and ensure that the result is always somewhere between the current PCC value and the target, which means that it never goes out of bounds.
The down side of this is that all functions that you call directly and all PC-relative globals must be within 2 GiB of each other instead of 4 GiB.
I believe this is the right choice for Zcheri as well, but it's probably hard to convince RISC-V folks to change the behaviour of a core instruction.
As a compromise, Zcheri explicitly permits this to be redefined by other extensions, which we do.

We also provide an `AUICGP` instruction, which allows a similar large-offset for relocations relative to our global pointer.
This instruction is rarely used (very few compartments have more than 4 KiB of globals), but the sequence to add a large displacement to globals can be quite long without it.
Again, this is specific to our ABI and compartmentalisation model and so is probably not useful to the broader Zcheri audience.

CHERIoT provides a temporal safety mechanism that is built from three parts:

 - A shadow memory that allows regions to be marked as freed.
 - A *load filter* that prevents loading pointers to freed objects.
 - A *revoker* that removes dangling pointers from memory.

We are confident that this design can scale quite nicely to tens of MiBs of RAM and perhaps four (maybe slightly more) cores.
It will *definitely* not be the right approach for 128-core servers with TiBs of RAM.
As such, it doesn't make sense to have this in Zcheri (though I'd hope that a future Zcheri standard can build on some of our prior work on CHERI+MTE compositions that would enable the same abstract machine in large systems).

The current Zcheri drafts assume the existence of an 'omnipotent' capability.
A capability that allows read, write, and execute is important for 100% backwards compatibility with APIs such as `mmap` (whether this is desirable is an exercise for the reader), but on CHERIoT we do not have such APIs.
CHERIoT has a *multi-rooted* capability hierarchy.
We have a read-execute root, a read-write root, and a sealing root.
These all have different permission encodings and so it is not possible to construct a write-execute capability (though you can have two capabilities to the same memory, one with write and one with execute permissions).

The remaining differences are fairly small.
For example, we have a set-bounds instruction with a large immediate.
This is important because our ABI sets bounds on globals.
This matters less on the larger systems for two reasons.
First, they typically use an ABI where pointers to globals can be loaded via a cap table, so bounds are not typically applied dynamically.
Second, larger systems have a different distribution of sizes for globals and so the 12-bit immediate is less useful (it hits almost 100% of cases for embedded systems).

The fact that CHERIoT is based on RV32E and not RV32I is not a difference: The Zcheri family can be applied to RV32E, RV32I, or RV64I.
The code size benefits for embedded software from 32 more registers is small, but the extra latency on cross-compartment calls and context switches with in-order architectures (and the extra space required) is much larger.
There is no value in 32 registers for us, but on a large application cores (especially large systems that wants to emulate x86-64) would make a very different choice.

What happens after 2.0?
-----------------------

2.0 is not the end for CHERIoT.
There are a lot of potential code-size optimisations.
Several of the compressed instructions are optimised for patterns that are less common in CHERIoT systems and the converse is also true.

The core model is now working well, but there's a lot of scope for both performance and code-size improvements.
We expect that most of these will be used by the compiler and have no impact on software that isn't writing hardware.

We will keep adding to the CHERIoT ISA, but after 2.0 I hope that we'll be able to maintain binary as well as source compatibility.
We will definitely provide source compatibility back to 1.0 for the foreseeable future.
