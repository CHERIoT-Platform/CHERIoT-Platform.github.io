---
layout: post
title:  "Rust coming to CHERIoT!"
date:   2025-11-21
categories: rtos publication
author: "David Chisnall"
---

The recent [UKRI press release](https://www.ukri.org/news/21-million-backing-for-technology-to-stop-cyber-attackers/) announcing £21M for CHERI projects includes two CHERIoT-focused activities.
The tools programme, in particular, has funded SCI Semiconductor to bring Rust support for CHERIoT to production quality.
This is being done in close collaboration with the folks from the University of Kent, who previously implemented Rust support for Arm's Morello (CHERI) platform.

The funding was awarded back in September, but the embargo was lifted last week and so we can talk about it publicly.

# Rust and CHERIoT have complementary benefits

I've [written previously about why safe languages and CHERI are complementary](/cheri/myths/2024/08/28/cheri-myths-safe-languages.html).
Superficially, both Rust and CHERI provide similar benefits in terms of memory safety.
That similarity goes away when you look at the details.

Rust provides a very rich set of type-system guarantees.
If you are writing software, you can use Rust's type system to enforce a range of properties that go beyond simple memory safety.
The key part here is 'if you are writing software'.

The industry learned some important lessons from Java and JavaScript attempts at language-level sandboxing: it's *very* hard to write a compiler that assumes that the programmer is an adversary.
Any soundness issue in the underlying type system or any bug in the compiler can be a security vulnerability if you assume that the person writing the software is malicious and actively trying to break the guarantees that the language aims to enforce.

The Rust compiler is currently tracking [107 bugs marked as soundness issues](https://github.com/rust-lang/rust/issues?q=is%3Aissue%20state%3Aopen%20label%3AI-unsound).
A typical Rust programmer is unlikely to encounter these.
Encountering these bugs typically require poking at corner cases of the language that you're unlikely to hit by accident.
In contrast, a malicious programmer wanting to insert a supply-chain vulnerability into something that you consume has a rich set of tools.

The CHERIoT compartmentalisation was designed with this kind of adversary in mind.
It assumes that you may be incorporating arbitrarily buggy or malicious code into a device's firmware and need to be able to protect against components that are compromised.
The checks that CHERIoT does at run time are less rich than those that Rust enforces at compile time, but are not bypassable, even with `unsafe` code or inline assembly.

This means that you can use Rust's type system to ensure that *your* code has strong confidentiality, integrity, *and availability* guarantees, while simultaneously using CHERIoT to ensure that supply-chain code cannot violate these guarantees (at least with respect to confidentiality and integrity, though availability is a bit harder).
Rust's guarantees make achieving high levels of confidence in your code much easier than if you used C/C++.
Rust is one of the few languages that delivers this kind of guarantee and is also able to run on small embedded devices (such as CHERIoT implementations), making it a very exciting choice for future CHERIoT development.

# CHERI and Rust give you the benefits of Rust faster

Rust gives rich guarantees to Rust code.
Most software is not written in a single language, and especially not in a relatively young language.
What if you have some legacy C/C++ component?
You can use it from Rust, but in most systems this means bugs in the legacy code can completely undermine the security guarantees of the Rust code.
A memory-safety bug in C code can corrupt *any* Rust object in the same process.

On a CHERI system, you get different levels of guarantee depending on how you mix languages.
Calling C code from Rust within the same compartment requires the C code to follow the CHERI rules.
If Rust code passes a pointer to C, the C code may tamper with objects reachable from there, but can't arbitrarily affect the system unless it has a much narrower set of bugs.
Stack uninitialised use bugs may cause C to access objects left on the stack, for example.

If you put the C code in another compartment, these guarantees become a lot stronger.
C code can tamper with objects passed as arguments (or objects reachable from them) but has no access to the Rust compartment's stack or globals.
This makes it *much* easier to reason about the impact of bugs in C code when adopting Rust.

You can take this even further and restrict the permissions on pointers passed from Rust to C.
Do you want C code to be able to read a Rust object or object graph?
Or perhaps modify an object but not capture a pointer to it (similar to borrow semantics)?
The hardware can enforce these properties.

This lets you get all of the great benefits of Rust from the *very first function that you write in Rust*, rather than having to wait until you've rewritten everything in Rust.

# What are we trying to do?

Initially, we aim to make Rust work as a source language for targeting CHERIoT.
CHERIoT is an embedded platform, so the no\_std + alloc mode for Rust (you can dynamically allocate memory, but can't use most of the standard library) makes sense as a target.
This will make it easy to port existing embedded Rust code to CHERIoT and to write new Rust components.
As the project runs, there will be a lot of quality-of-implementation work to ensure that it's in a state to upstream and, until then, in a state where we can support it.

The next step in the project is to make sure that Rust is a first-class citizen of the CHERIoT platform.
We have a set of C/C++ language extensions to provide rich compartmentalisation features.
Rust will need equivalents of these.
A direct port of the features is quite easy and works as a minimum viable product, but we'd also like to make sure that these work as *idiomatic* Rust: you shouldn't have to write C-like Rust to use CHERIoT features.

Finally, there are several things in the Rust type system that can be dynamically enforced in CHERIoT.
In current Rust, every call to non-Rust code must be `unsafe`.
I hope that we'll be able to relax this requirement and have the compiler enforce Rust properties by removing permissions from pointers before calling C.

There are a lot of subtleties here.
The authors of Tock [discovered that exposing Rust to untrusted code has some issues](https://tockos.org/assets/papers/2025-sosp-tock-decade.pdf) and provided solutions that worked in their specific context.
I'm optimistic that the richer substrate of the CHERIoT ISA gives the compiler more tools to provide generic solutions to these issues.

The funded project is specifically scoped to CHERIoT, but we're building on work from Morello and hope to make it easy to support other CHERI platforms.

# Rust enables verification

The [Verus](https://github.com/verus-lang/verus) project builds language-integrated formal verification tools in Rust.
This is particularly interesting for core parts of CHERIoT RTOS.
The lowest level parts of an operating system are tricky for safer systems languages because the things that they do are *intrinsically* unsafe.
They must be able to use escape hatches that opt out of parts of the safety guarantees of a language like Rust *because they are the things that implement those guarantees*.

You can think of rich type systems as *off-the-shelf* verification tools (they define some generic properties and prove them for every program, and reject those for which they can't prove them), whereas the lowest-level parts of systems code need *bespoke* verification to prove one-off sets of properties.
There was [some great work at SOSP this year showing how Verus can prove key aspects of a kernel](https://mars-research.github.io/doc/2025-sosp-atmo.pdf) and I'm excited to see how far we can get with this in CHERIoT.

# Where will this project live?

The [CHERIoT Rust project has its own web site](https://rust.cheriot.org) which is where we'll post more public information.
The [compiler repository](https://github.com/CHERIoT-Platform/cheri-rust) is public, but is not yet ready for general use.
We'll post calls for testing when it reaches an early preview state.
As with any other aspect of CHERIoT, contributions are welcome.
Please join [our public Signal chat](https://signal.group/#CjQKIElxAs3t3MUEMOEmQEuMHRK4rErUk2xVeFzjAjFXAShzEhCK9qQwEMFKGLGZnCjrQ7zm) if you want to discuss the project or help out!
