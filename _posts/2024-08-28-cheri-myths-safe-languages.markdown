---
layout: post
title:  "CHERI Myths: I don't need CHERI if I have safe languages"
date:   2024-08-28
categories: cheri myths
author: David Chisnall
---

There's a recurring myth that CHERI and safe languages are solving the same problems and that, if you have one, you don't need the other.
When I joined the CHERI project in 2012, my primary motivation was producing hardware that enabled safe interoperability between languages, so I never felt that safe languages and CHERI were in tension.
Quite the reverse: My work on CHERI was motivated by a desire to enable safe language adoption and the combination of CHERI and safe languages is far more powerful than either in isolation.

CHERI and safe languages provide very different benefits.
Each have their strengths and weaknesses and, in this post, I'll try to explain how their strengths combine.

## A single memory-safety bug can ruin your entire day

The [WannaCry ransomware attack](https://en.wikipedia.org/wiki/WannaCry_ransomware_attack), which cost billions of dollars, was enabled by a single use-after-free bug.
The [CrowdStrike incident](https://en.wikipedia.org/wiki/CrowdStrike#2024_CrowdStrike_incident), which also cost billions of dollars, was the result of an uninitialised-use bug (compounded by a lot of operational failures).

Memory-safety bugs are particularly bad because they step outside of the language's abstract machine.
After a program passes a memory-safety bug, it may do anything.
It may access objects that should not be reachable, modify immutable objects, and even execute instructions out of sequence to do things that are not part of the source-code description of what the program should do at all.

A single memory-safety bug can be all that an attacker needs to get an arbitrary-code execution exploit.
Rewriting 99% of a program in a safe language but leaving just one memory-safety bug does not prevent at-scale deployment of malware.

This was one of the problems that CHERI aimed to solve, in two ways.
At the fine granularity, CHERI offers language-agnostic memory safety primitives.
Compilers can generate, or humans can write, assembler making use of these primitives to capture and then enforce memory safety of source-level objects.
The switcher is the most privileged part of a CHERIoT system and is written in assembly (it has to do things like save all registers, which are not possible in any high-level language).
In spite of its privilege, it remains bound by the memory-safety rules.
It cannot access any compartments other than the caller and callee.
It cannot access any memory other than that reachable from the registers when it is invoked and its internal state, unlike a traditional OS kernel that can access anything.

Enforcing memory-safety guarantees in the hardware makes it possible for safe-language code to call unsafe or, most excitingly, _differently_ safe code and yet still enforce source-level properties.
Consider a single program composed of code written in all of Rust, Java, and Haskell.
Each is in the 'safe languages' category, but provides different guarantees.
Rust code passing a borrowed reference to Java may wish to ensure that it isn't captured, a property that CHERIoT can enforce directly (or CHERI can enforce with indirection).
Haskell passing an object to Java or Rust may wish to guarantee deep immutability, a property that CHERIoT and all Morello or later CHERI platforms can enforce.

When a CHERI CPU raises a trap, it happens *before* any data has been leaked or corrupted as a result of the memory-safety violation.
This means that it's at a place where you can recover.
For example, a JVM attempting to write to an immutable object borrowed from Haskell can raise an exception in Java.
The Haskell component doesn't need to know how Java handles dynamic failure, it just knows that nothing can modify its immutable objects.

At a coarser granularity, CHERI makes it easy to sandbox entire unsafe components.
You can take a C library and put it in an isolated compartment.
A lot of attempts have been made to build sandboxing technologies that are independent of memory safety but they have not been widely adopted because they don't lend themselves to a simple programmer model.
Programmers think about memory in terms of objects with pointers between them, not in terms of pages mapped in a linear address space.
CHERI lets you share objects between compartments, without having to think about anything lower level than the language abstract machine.
You can use CHERI to share a read-only view of an object graph, a write-only view of a buffer, and so on.

In the CHERIoT network stack, we take advantage of this between the TLS and TCP/IP compartments.
The TLS stack maintains its own ring buffers for messages.
When it calls into the TCP/IP stack to receive some data, it passes a bounded capability to a subset of the buffer, with write-only, no-capture permissions.
By the end of the call, the TLS stack knows that the TCP/IP stack doesn't have access to the buffer and can't have leaked any stale data in there.
The TLS compartment contains BearSSL, which is mostly written in a domain-specific language that compiles down to C for side-channel-resistant crypto routines, but the same guarantees are available to any language that can target the platform.

## Rewriting large codebases is rarely the correct answer

There's a pervasive narrative in some camps that rewriting everything in a safe language is the path to fixing security.

### Rewriting comes with a large opportunity cost

Just looking at open source code, there are around ten billion lines of C/C++ code.
It's not clear how much more exists in proprietary projects.
Optimistically, the cost of rewriting all of this is hundreds of billions to trillions of dollars.

That's a lot, but most importantly, it's investment that takes away from writing new code.
If one team decides to spend a year rewriting a new project and another spends the year writing new code that adds new features, which do you think will be more successful in the market?
Rewriting code with no user-visible benefits is not likely to drive sales.

Writing new code that solves new problems, of course, doesn't come with opportunity cost concerns.
If you can write *new* code in a safe language, especially a safe language that improves developer productivity, then that's an obvious win.

### Rewriting can introduce new bugs

Will a team that rewrites a legacy codebase or one that incrementally improves one have fewer bugs in their codebase?
If you rewrite existing code in a safe language then you can guarantee, by construction, that the code does not have bugs in the category that the language enforces.
The value of this can't be understated.
Knowing that code that compiles is free from a particular class of error has a huge value.

In spite of this improvement, there remains a big difference between 'memory safe' and 'correct'.
Even formal verification doesn't guarantee correctness, only that the properties that are checked hold as long as the axioms hold.
In general, new code is more likely to contain bugs than old code (which people have tested and fixed bugs in, in production, often over many years).
We [saw this with Microsoft's confusingly named sudo for Windows](https://www.tiraniddo.dev/2024/02/sudo-on-windows-quick-rundown.html): It was written in Rust, but the lack of memory safety bugs did not prevent it from being a poor design for security.

Again, writing *new* code in a safe language is likely to reduce the number of new bugs that you introduce relative to something like C and so is often a better choice.

### Rewriting may not be possible

All of this is assuming that rewriting is possible.
A lot of TCB code is intrinsically unsafe.
A memory allocator, for example, defines the notion of a heap object and so must sit below the abstraction of any notion of memory safety.
The same is true of any part of an OS kernel that has views of userspace memory.
These can definitely be made *safer*, but not safe.

Conversely, some code is intrinsically safe.
Some C codebases have never had a memory-safety issue because they simply don't do any pointer arithmetic or dynamic allocation: they work with pre-allocated fixed-sized structures.
Rewriting these in a memory-safe language is often trivial but yields few benefits.

### Rewriting requires specialised skills

Consider something like the FFMPEG library's libavcodec.
This has had a lot of optimisations implemented over the years, which rely on intimate knowledge of the problem domain.
There are almost certainly people who could rewrite it in Rust and get at least equivalent performance.
Hiring an experienced Rust programmer is hard (though getting easier), hiring an expert on video and audio CODECs is very hard (and not getting easier), hiring someone with skills in both is incredibly hard.

The same applies to things like crypto libraries, where it's very easy for an experienced programmer who is not a domain expert to introduce vulnerabilities.
This may be in the form of timing side channels, or more subtle things such as choosing keys that are vulnerable to particular known attacks.

This doesn't always apply.
For more common tasks, the metaprogramming facilities in a higher-level language may make the rewrite significantly simpler to write and maintain than a C original.


## CHERI does not fix bugs for you

CHERI doesn't guarantee that your code is free from memory-safety errors, it guarantees that any memory-safety bugs will trap and not affect confidentiality or integrity of your program.
With compartmentalisation, components can be isolated into separate failure domains, which improves reliability a lot.
For example, a memory-safety bug in our TCP/IP compartment will trigger a reset of that compartment.

This is a much weaker guarantee than most safe languages provide.
A lot of things that, in C/C++/assembly code will trap on CHERI, will cause compile failures with Rust or Ada.
If they cause a compile failure, then you have to fix them before you ship your code and that guarantees that they don't exist in production at all.
This is obviously a benefit.

This benefit isn't limited to memory safety.
The more properties you can check at compile time, the lower your maintenance costs are likely to be.
At some point, the cost of all of those checks becomes too large (in terms of writing them, modifying them as the codebase evolves, and compile time) but there are often a lot of simple properties that can be easily checked.
Rust or modern C++ make it easy to check a lot of things at compile time and enforce properties of types that go well beyond memory safety.

CHERI protects you against bugs in your code doing damage, combining that with eliminating other categories of bugs is a win.

## Your supply chain may not be memory safe

One of the big arguments in favour of Rust (and one that I have personally used) is that an `unsafe` keyword is a big flag in code review.
This is a *much* better place to be for your own code than C, where anything that involves a pointer may be a memory safety bug unless you read all code that touches the pointer or other aliases of it.
When you're writing code with a set of non-malicious collaborators, this is a big benefit.

But what happens with the rest of the supply chain?
The [Rudra paper](https://dl.acm.org/doi/10.1145/3477132.3483570) used some static analysis to find 264 memory-safety bugs in cargo crates, and issued 76 CVEs and 112 RustSec advisories as a result.
These were difficult to spot unless you knew the exact incorrect idiom to look for, so a cursory glance at `unsafe` code in your dependencies would not have helped.

This is the best case.
The bugs that Rudra found were accidental.
The [cve-rs project](https://github.com/Speykious/cve-rs/tree/main?tab=readme-ov-file) shows how to exploit a soundness issue in the Rust borrow checker to implement any kind of memory-safety bug entirely without `unsafe`.
Even a type-safe language may have implementation bugs that make it possible to sneak in malicious code.

This is not just a problem for Rust.
Go is a garbage-collected type-safe language, but it exposes a slice type that contains a base, length, and capacity.
This is not atomically updated and so you can have two goroutines race storing to a shared variable that is a slice type and a third will eventually see the base of one and the length of the other, allowing out-of-bounds accesses, which can then be used to escape type safety.
The Java VM is infamous for security vulnerabilities that allow programs to break type safety in various ways.

The more complex a type system is, whether it's enforced statically or dynamically, the more likely that there will be a bug that allows an attacker to exploit it.
When you're pulling in a million-line dependency, auditing it to make sure that it doesn't trigger such a vulnerability is impossible.

Java shipped with a lot of complex infrastructure for sandboxing components.
The [`SecurityManager`](https://docs.oracle.com/javase/8/docs/api/java/lang/SecurityManager.html) let you restrict the privileges of running code and elevate privileges later.
This depended on the correctness of most of the rest of the JVM.
In Android, Google removed it (technically it's still there, you just got an exception if you installed a non-default policy) and made the process boundary the only security boundary.
They did this because anything that depends on the correctness of the entire JVM for security is likely to be broken.
The same rationale should apply to compartmentalisation models that rely on other languages' type-safety guarantees.

CHERI does not have this complexity problem.
The security properties of a CHERI ISA are simple and local.
They have been formally verified at the ISA level on two CHERI variants and the CHERIoT Ibex core has had its implementation formally verified.
This is possible because (slightly oversimplifying) the properties are easy to understand:

 - All memory accesses require a capability with the right permissions.
 - Capability permissions may not increase.

Beyond that, the TCB code that enforces higher-level properties such as compartment isolation, thread isolation, and no-capture guarantees (in combination with hardware) is only around 300 instructions on CHERIoT and so should also be amenable to formal verification.

This means that you can have strong run-time guarantees even when using malicious third-party code that exploits bugs in a type checker for a safe language.
These guarantees also apply to binary-only components, including ones written in unsafe languages.

CHERI compartmentalisation does not just restrict the damage from memory-safety issues.
The [`cheriot-audit` tool](https://github.com/CHERIoT-Platform/cheriot-audit) lets you audit exactly which things a compartment can do outside of itself in a CHERIoT firmware image (which functions it can call in other compartments, which pre-shared objects and memory-mapped I/O regions it can access and with which permissions).
You can reason about the damage from a compromise even if an attacker can gain arbitrary-code execution in a compartment.
For supply-chain security, you should assume that a third-party component is compromised and includes malicious code.
CHERIoT lets you reason about what it can do in these cases.
In contrast, if a Java or Rust component is malicious and uses (intentional or otherwise) unsafe language features, it can to anything that the program can do.

## The future should be safe languages on CHERI

CHERI and safe language give different sets of guarantees, but those guarantees *compose* to be stronger than either in isolation.
CHERI makes it easy to safely interoperate with untrusted code.
Safe languages make it easier for your trusted code to be *trustworthy*.
Safe languages with CHERI make it easy to interoperate between safe and unsafe languages, without compromising the guarantees of *any* language.

Folks at Kent [have ported Rust to CHERI targets](https://www.cs.kent.ac.uk/people/staff/mjb211/rust/index.htm) and others at [AdaCore have ported Ada](https://www.adacore.com/uploads/techPapers/Elevate-Security-Confidence-with-Memory-Safe-Hardware-and-Software.pdf).
We hope to have both of these on CHERIoT over the next year.
Being able to generate code that runs in a CHERIoT compartment should be fairly simple but there's additional work to make it easy to define and use compartment entry points and so on.

CHERIoT even comes with a [JavaScript interpreter](https://microvium.com) that preserves JavaScript type-safety properties even when C code takes pointers to JavaScript objects.
Using CHERI doesn't mean you have to stick with unsafe languages, it means that your transition to safe languages can be motivated by the benefits of those languages, not by the shortcomings of C.
