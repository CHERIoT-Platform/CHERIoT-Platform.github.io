---
layout: post
title:  "CHERIoT at SOSP 2025"
date:   2025-10-18
categories: rtos publication
author: "David Chisnall"
---

<img alt="Presenting the CHERIoT RTOS paper" width="50%" style="margin-left:auto;margin-right:auto;display:block" src="/images/2025-10-14-sosp.jpg" >


This week, some of the CHERIoT team were at [The 31st Symposium on Operating Systems (SOSP 2025)](https://sigops.org/s/conferences/sosp/2025/index.html) presenting the first paper about the CHERIoT RTOS:
{% cite 10.1145/3731569.3764844 %}.
This paper describes the CHERIoT RTOS and how it builds on the ISA features to deliver fine-grained compartmentalisation, easy programming, and a tiny trusted computing base (TCB).
I also gave a keynote on how CHERI impacts operating system design for the [KISV workshop](https://kisv-workshop.github.io) associated with SOSP.
There were a lot of good discussions, and I hope to see more folks looking at CHERIoT RTOS.

It was interesting to compare our approach with Tock OS, which remains the gold standard for security on non-CHERI embedded devices.
One of the papers in the same session as ours discussed the problems Tock has with untrusted code in userspace violating the Rust invariants.
A lot of these are intrinsic to the problem of interfacing a language that provides (and can therefore depend on) a very rich set of compile-time properties with one that does not guarantee any of these.
It was particularly nice to see that the CHERIoT ISA allows CHERIoT RTOS to enforce some of these properties (such as non-aliasing arising from a no-capture guarantee) *even across trust boundaries*.
That makes me optimistic that CHERIoT RTOS will be one of the best embedded targets for Rust code (more on this coming soon!).

Full citation
-------------

{% bibliography --cited %}

