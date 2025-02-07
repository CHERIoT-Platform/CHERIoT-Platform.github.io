---
layout: post
title:  "I don't care about memory safety"
date:   2025-02-07
categories: philosophy
author: David Chisnall
---

This is a repost of something I wrote on LinkedIn a year and a half ago, but it turns out no one reads LinkedIn, so I'm reposting it here.

This might seem a strange thing for someone who has been working on hardware memory safety for over a decade to say, so perhaps I should clarify.
I care about memory safety in the same way as I care about addition.
If it works, I can build interesting things on top of it.
In fact, most of the interesting things that I want to build rely on it as a core foundation block.
If addition doesn't work, I have no way of reasoning about anything in a program.

The same is true of memory safety.
To me, the fact 70% of security vulnerabilities arise from a lack of memory safety is not the reason that memory safety is important.
It's important because a single memory-safety bug can completely undermine all of the guarantees that I want to rely on.
An out-of-bounds access or a use-after-free bug can leak or corrupt arbitrary state anywhere in my program.
If I think something is private to my thread because nothing in my program stores a reference to it anywhere that is reachable by another thread, that is true only as long as my program is memory safe.
If I think an object is immutable because I don't expose any APIs that modify it and my type system says it's immutable, that's true only as long as my program is memory safe.

As soon as my program contains a single memory-safety error, none of these properties hold, even if you have formally verified some of them.
The [EverCrypt](https://www.microsoft.com/en-us/research/publication/evercrypt-a-fast-veriﬁed-cross-platform-cryptographic-provider/) project did phenomenal work producing formally verified (including side-channel resistant) cryptography libraries but a formally verified program only has the properties that are proven if the axioms hold.
If you don't have memory safety then these axioms don't hold.

If you *do* have memory safety, then you can start to build interesting things.
I started working on memory safety because I wanted rich component systems.
In the '90s, various platforms provided rich component models.
Word documents could embed COM controls that embedded other rich applications.
Most of these things went away because running arbitrary code from a third party is dangerous.
About the only programs that do it safely are web browsers (which are designed to do it and nothing else).

In 2005, I attended a talk by Alan Kay.
His big reveal in the middle was that his slides were not actually a PowerPoint presentation, they were a Smalltalk program.
In the middle, he drew some insects and then wrote some code that made them run around the edges of embedded images and videos.
He asked a reasonable question: 'Why would you use a program that wasn't a programming language?' Unfortunately, if you exchange documents, this means running arbitrary code from other people.
Imagine being able to pull code from the Internet and embed it in a document that you send to someone else, without worrying about it compromising your (or their) system.

You can start to do some of these things with WebAssembly, but then you run into a universal problem:

**Isolation is easy, (safe) sharing is hard.**

We know how to do safe isolation.
It's been used to protect nuclear launch systems for decades.
You have a computer that isn't connected to other computers.
And then you put it in a locked room.
And then you put people with guns outside and tell them to shoot people who try to get in and shouldn't.

For lower-stakes systems, you can skip the people with guns and possibly even the locked room.
This is why memory management units (MMUs) on commodity hardware changed the world.
Multiple programs could run and a bug in one would not crash another (or the whole system) but they could access the same set of files.
Two users could share the same system and a kernel could give them access to both private and shared sets of files.

The lack of memory safety is why we can't have nice things.

This is why I don't get excited by memory safety that comes with asterisks.
Asterisks like 'as long as no one guesses a 4-bit number' (MTE).
Or, if you need to fake three pointers for a full compromise, as long as no one guesses a 12-bit number.
So, if you have a Windows or Android memory-safety bug, you only get to compromise one in 4,096 users, giving you around five million systems (assuming that you only get one attempt), and you'll probably be detected on some of the ones that you don't compromise.

Or asterisks like 'as long as you don't use the unsafe keyword, unsafe package, or sun.misc.Unsafe package' (Rust, Go, Java), or 'as long as you don't use any code from unsafe languages' (all memory-safe languages).
I want to use unsafe-language code! GitHub has over thirteen *billion* lines of C/C++ code that I really, really don't want to rewrite (or pay for someone else to rewrite) in a safe language.

I want to be able to reuse that code, but confine it such that bugs in it are limited.
I want to be able to call into it knowing that it can write through pointers I pass, but can't access anything that isn't shared.
I want to know that it can't access anything in my process (let alone elsewhere on my computer) unless I explicitly gave it the rights to.
I want to not care if it has bugs because I can explicitly limit the blast radius without the spooky action at a distance that memory safety bugs imply.

This is the kind of system that [CHERI](https://en.wikipedia.org/wiki/Capability_Hardware_Enhanced_RISC_Instructions) was designed to let me (and everyone else) build.

It's great that we can reduce the number of memory-safety bugs, or make them harder to exploit.
Rewriting code in safe languages eliminates classes of failures and makes the world a better place, if you can afford the opportunity cost of rewriting then please do! Deploying mitigations that mean an attacker gets control of five million machines instead of giving them control over two billion vastly reduces the damage that they can do.
These approaches don't let me build the kinds of systems that get me excited though.

Most of the [published work on CHERI](https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/cheri-publications.html) has been about running existing software.

The fact that [we can run an entire modern POSIX-compliant kernel and userland and a load of software with memory safety](https://www.cl.cam.ac.uk/research/security/ctsrd/pdfs/201904-asplos-cheriabi.pdf) is important for adoption, because that's the software that exists in the world now, but it's underselling what you can do if you are able to treat memory safety as something that just works, in the same way that integer arithmetic just works (and far more comprehensibly than the way that floating point arithmetic works).

With CHERIoT, we've started to show a glimpse of this world.
You can take existing C/C++ code and recompile it to run in a CHERIoT compartment.
You can take for granted the fact that any out-of-bounds access and any use-after-free will trap.
And that means that you can share an object with another compartment by passing it a pointer.
You can rely on the fact that, if you pass another compartment a pointer to an on-stack object (or any pointer that you've explicitly marked as ephemeral), any attempt to capture that pointer will trap.
You can share a read-only view of a buffer, by providing a pointer without write permission, or of a complex data structure by providing one without transitive write permission.

You can use what [Robert Watson](https://www.cl.cam.ac.uk/~rnw24/) refers to as the 'vacuum-cleaner model of software development': point your vacuum cleaner at the Internet, vacuum up all of the components you need, and ship them.
Only now you can audit exactly what each of those third-party components has access to.
Because even assembly code has to follow the core rules for memory safety, you can write policies about what they should be able to access and audit them before you sign a firmware image.
Just guessing an address that contains some important data or some code doesn't let you access that memory.
You can write secure code without having to worry about entire classes of errors and you can avoid having to deploy costly security updates for components that are sandboxed.
Most importantly, you can *understand* security properties of a piece of code by looking at its interface.

CHERIoT targets small systems because it's feasible to replace the entire OS with something that provides abstractions that are vastly more secure *and* usable than anything that's possible with existing hardware.
Even there, we've only started to scratch the surface of what is possible when memory safety (including referential integrity and control-flow integrity) are just things that you are able to just assume when building the rest of the system.
It will probably take a at least decade between commodity CHERI hardware shipping and consumer operating systems being able to rely on it for core abstractions.

*That* is what gets me excited: being able to deeply embed untrusted code, with rich communication between it and other bits of untrusted code, within complex systems.

I hope to see you all in that world.
