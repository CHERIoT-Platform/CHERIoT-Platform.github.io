---
layout: post
title:  "Formally verifying security properties of CHERI processors"
date:   2024-08-02 14:00:00 +0100
categories: formal verification security
author: "Anna Duque Antón and Johannes Müller"
---

In 2018, the two infamous attacks Spectre and Meltdown raised awareness of timing side channels in the microarchitecture of modern processors.
In the following years, ensuing research efforts have been made by academia and industry to investigate and mitigate these findings.
So far, the focus has been on larger cores with powerful performance optimizations such as out-of-order execution and speculation.
But even small processors optimized for low-power environments and featuring in-order pipelines can be vulnerable to these attacks.
We have used our new formal verification framework *VeriCHERI* to detect a vulnerability to a Meltdown-style attack in CHERIoT Ibex.
This was [reported](https://github.com/microsoft/cheriot-ibex/issues/38) and [fixed](https://github.com/microsoft/cheriot-ibex/commit/876a46ab0746e7b6d7a31a72eb834737d8076aca) in May.

### What makes a timing attack *Meltdown-style*?

The classical Meltdown attack exploits the out-of-order execution of processors to read from protected memory locations.
In a Meltdown attack, the attacker can run code on the processor but has no access to certain memory locations, like the kernel address space.
The key aspect is that the out-of-order execution creates a transient time window that allows an early execution of an (illegal) load access.
This creates a race condition between the illegal access and the access permission check, which raises an exception and triggers corrective measures.
In a functionally correct implementation of the processor, the illegally accessed data does not affect the architectural state of the processor, like the register file or the PC.
However, it may leave a footprint in the microarchitecture, for example in the cache, to be extracted by the attacker.
Race conditions between the effect of an illegal memory access and the access control checks preventing the access do not require features like out-of-order execution.
*Meltdown-style* attacks can already be created by simple in-order load instructions.
Out-of-order execution merely amplifies the effect by increasing the size of the transient time window.

The vulnerability we detected in CHERIoT Ibex fits into this category.
Consider a situation where the PCC restricts fetching instructions from outside the current compartment.
Executing a jump instruction, i.e., fetching an instruction from an address outside of the PCC bounds, results in an exception and a flushed pipeline.
However, if the 32 bits contain unaligned instructions, e.g., due to compressed instructions, and the upper halfword contains the lower 16 bits of an uncompressed instruction, the processor uses an additional cycle to fetch the rest of the instruction, before the access bounds are evaluated and an exception is triggered.
Whether or not the upper halfword contains an unaligned and uncompressed instruction is determined by two specific bits of the instruction (bits ``[17:16]``).
VeriCHERI, our new verification method, discovered that a potential attack exploiting this vulnerability can probe any word in the memory for these two bits.
The probing result is then contained in the overall execution time and in the value of a performance counter.

The CHERIoT development team fixed this vulnerability by adjusting the fetch FIFO to always behave as if there was an unaligned compressed instruction (``rdata[17:16] != 2'b11``) when there is an address bound violation and the current fetch PC is unaligned.

### How did we detect this vulnerability?

VeriCHERI is a new formal verification framework targeting security vulnerabilities in CHERI-enhanced processors.
The key idea is that we start from abstract security requirements targeting confidentiality and integrity.
Based on these general notions of security, we formulate security properties for the microarchitectural implementation.
This is a significantly different approach compared to previous verification methods, which focus on verifying that the design conforms to a specification.
VeriCHERI allows us to target not only security violations due to functional bugs, but also Meltdown-style timing side channels such as the one described above.
At its core, VeriCHERI consits of only 4 security properties; these can be checked using the power of commercial property checking tools.
Verification times for CHERIoT Ibex range from a few seconds to 31 minutes for detecting vulnerabilities in the original versions or to prove that the fixed design is secure.
We refer interested readers to our [paper](https://arxiv.org/abs/2407.18679) about VeriCHERI {% cite vericheri2024 %}.

Full citation
-------------

{% bibliography --cited %}
