---
layout: post
title:  "Removing the AUICGP instruction"
date:   2026-03-23
categories: isa toolchain
author: David Chisnall
---

> Perfection is achieved, not when there is nothing more to add, but when there is nothing left to take away
> 
> – Antoine de Saint-Exupéry

In the last week or so, we've started landing patches in our compiler to remove the `auicgp` instruction.
This is something that we'd hoped to do for a while because we want to remove the instruction entirely in CHERIoT v2 (to be based on the upcoming RV32YE standard).
This post will explain why we had it, why we wanted to get rid of it, and how we've removed the need for it.

What is auicgp?
---------------

In CHERIoT, every global is either accessed relative to the program counter (PC) for code or immutable globals, or the global pointer (GP) for read-write globals.
On RISC-V, PC-relative addressing is done via the `auipc` instruction, which takes a 20-bit immediate, left shifts it by 12, and adds it to the PC.
This is used paired with a load, store, or add (if you just want to materialise the address as a pointer) to fill in the remaining bits.

For symmetry with base RISC-V (more detail on this later), we introduced a similar instruction that added a shifted immediate to the global pointer: add upper immediate to (capability) global pointer (`auicgp`).
Originally, because we're based on RV32E, we used one of the bits in the destination-register field to differentiate this from `auipcc` instruction, so it fitted into some spare encoding space.

In the initial public release, we changed this because RISC-V promises never to use that bit of encoding space for standard extensions and we hoped to standardise it.
Since then, the `auicgp` instruction has been the least popular bit of the CHERIoT ISA, for a variety of reasons.

First, it's an enormous instruction.
With a register target and a 20-bit immediate, it uses a full major opcode.
That's 1/128 of the total encoding space.
In practice, it's worse than that, because the RISC-V encoding requires 32-bit instructions to start 11 (01, 10, and 00 are reserved for 16-bit encodings), it's actually 1/32 of the space available for 32-bit instructions.
That's a lot to pay, though it might be worth it if the instruction were used a lot.

Unfortunately, `auicgp` is very rarely used.
Our ABI points the global pointer to the *middle* of the globals region, which means that we can use the entire 4 KiB range of the 12-bit immediate in loads, stores, and adds.
On RISC-V, the normal model is for the compiler to emit a long instruction sequence for operations like global accesses and then the linker to *relax* this by deleting instructions.
We followed this pattern for `auicgp`, with the linker deleting it if the immediate is 0.
As a result, almost all of the `auicgp` instructions were deleted.
Our original linker version didn't implement support for relaxations (upstream LLD didn't support them for RISC-V when we started working on CHERIoT!) and so we weren't initially sure how feasible deleting them would be, but we've had linker relaxations working for a few years now and they work very well.

The RTOS test suite contains a few `auicgp`s, but only because we've intentionally written tests to make sure that large globals work.
In most firmware images, there are no instances of the instruction that survive relaxation.
So we're using 1/32 of the encoding space for 32-bit instructions for an instruction that, to a first approximation, never gets used.

If that isn't bad enough, it's also annoying for simple pipelines to implement.
RISC-V is designed so that the source and destination registers are always in the same place in instructions.
Once you move to long pipelines, have register rename, or extensions with multiple register files (such as floating-point or vector extensions) then this is of little or no benefit.
But CHERIoT targets the kind of microcontroller design that RISC-V overfitted for.
Simple in-order pipelines find it useful to do register fetch early.
The `auicgp` instruction is the only one that has fetches from the general-purpose register file but does not have the registers in the correct location.
This adds extra multiplexing that, on a short in-order pipeline such as CHERIoT Ibex, impacts the critical-path length.

So, unfortunately, there is no situation in which this instruction is a good idea, except that it made it easy to create dense code via largely orthogonal code paths.
To understand why we incorporated it, you need to understand how RISC-V normally generates pointers to globals.

Why did we have auicgp?
-----------------------

Let's start with the simple case in RISC-V, of a static binary (how you'd normally compile embedded firmware) accessing a global.
A simple sequence loading an `int` from a global called `x` would look like this:

```
	lui     a0, %hi(x)
	lw      a0, %lo(x)(a0)
```

This has two relocations.
The load upper immediate instruction will materialise the top 20 bits of the address via the `hi` relocation, then the load will materialise the low 12 bits in its immediate from the `lo` relocation.
This is very efficient, but doesn't work on a CHERI system because you can't just make up an integer address and use it for loads and stores.

That's not a big problem, because it's the same issue that position-independent code needs to handle, and RISC-V does this with the same number of instructions.
When you're compiling position-independent code, RISC-V will instead generate the following sequence:

```
.Lpcrel_hi0:
	auipc   a0, %pcrel_hi(x)
	lw      a0, %pcrel_lo(.Lpcrel_hi0)(a0)
```

Again, this materialises the address in two instructions, including folding half of the address-materialisation into the load.
The difference is that the high bits are not created as an absolute address, they are created by adding a value to the current PC value.
The `%pcrel_lo` relocation looks a bit odd because it refers to the address of the `auipc` instruction, *not* the address of `x`.
This is because of limitations in ELF relocations.
The linker needs to know both the address of the `auipc` instruction (which doesn't *have to be* next to the `lw`, and might not be if the `lw` is in a loop) and of the target, but ELF relocations can't express both of these.
Fortunately, the linker can work around this by looking up the relocations that apply to the address of the `auipc` instruction to find the target.

On CHERIoT, accessing read-only globals works in *exactly* the same way.
They are stored within the bounds of the program counter for the current compartment (or library) and so we can compute a displacement and load.
Read-write globals are instead stored within a region reachable via the global pointer.
These capabilities don't overlap, and they have distinct permissions.

This is where we encounter the first problem.
Most of the time, the compiler knows whether a global is read-only or not and so knows whether it needs to emit a sequence relative to the program counter or the global pointer.
This is not universally true.
In particular, C inherited a notion of 'common linkage' from Fortran and this makes it possible for different compilation units to *disagree* on whether something is read-only and leaves it for the linker to figure it out.

For read-only globals, the sequence looks like this (note: the `ct.` prefix is the official vendor prefix for CHERIoT instructions):

```
.LBB0_1:
	ct.auipcc    a0, %cheriot_compartment_hi(x)
	ct.clw       a0, %cheriot_compartment_lo_i(.LBB0_1)(a0)
```

But for read-write globals, it looks like this:

```
.LBB0_1:
	ct.auicgp   a0, %cheriot_compartment_hi(x)
	ct.clw      a0, %cheriot_compartment_lo_i(.LBB0_1)(a0)
```

This hopefully makes it clear how the `auicgp` instruction is useful.
These two sequences look identical and so the linker can rewrite one to the other by simply rewriting the opcode of the first instruction.
If the compiler tries to do PC-relative addressing for a read-write global, the compiler can turn the `auipcc` into an `auicgp`, or vice versa.

For read-write globals, the linker can also do *relaxation*.
In the (common, almost universal) case that the address of `x` is within the displacement from the global pointer of a single instruction, the second sequence becomes this:

```
	ct.clw	a0, {displacement of x from the global pointer}(c)
```

Note that this is now *fewer* instructions than the baseline RISC-V case.
We hit this in a lot of cases, because a lot of global accesses are simple reads and writes of scalar values.
That's not universally true.
If the address of the global escapes the analyses that the compiler uses to prove that a dereference is in-bounds (which are currently fairly naïve, but will improve over time), then we must materialise a full capability.

This is why it's important to design a CHERI ABI along with a threat model.
Our threat model requires that any compartment be able to enforce a list of memory-safety properties *against untrusted code*, but treats things like memory safety for stack or global variables *within* a compartment as defence-in-depth properties.
Code within a compartment is allowed to access all globals for that compartment and so the threat model says it's fine to make the compiler responsible for enforcing those bounds.
But when a pointer to a global is passed to another compartment (possibly indirectly via another compilation unit, maybe in a different language, in the same compartment) then the compiler taking the address of that global *must* be able to apply bounds and permissions.

If we are taking the address of `x`, the compiler will generate a sequence like this:

```
.LBB0_1:
	ct.auicgp       a0, %cheriot_compartment_hi(x)
	ct.cincoffset   a0, a0, %cheriot_compartment_lo_i(.LBB0_1)
	ct.csetbounds   a0, a0, %cheriot_compartment_size(x)
```

Note that now the second part of the address isn't folded into the load instruction.
This doesn't hurt code size as much as you might think, because now we're likely to do offset-zero loads and stores on the result and RISC-V lets us express these as 16-bit instructions.

The new instruction is applying the bounds.
As before, the linker will *normally* relax away the first instruction, so we end up with this sequence for taking the address of a global:

```
	ct.cincoffset   a0, a0, {offset of x}(gp)
	ct.csetbounds   a0, a0, {size of x}(x)
```

Again, this is the same number of instructions as baseline RISC-V, but now with bounds applied.
For comparison, here is the RISC-V version:

```
	auipc   a0, %pcrel_hi(x)
	addi    a0, a0, %pcrel_lo(.Lpcrel_hi0)
```

There is one annoying case though: what happens if the bounds are too large for an immediate?
Here, we need an extra instruction (and register) to hold the immediate.
This means that our worst case (large bounds, address taken or not provably in bounds) is five instructions (if it takes two instructions to materialise the immediate for the bounds), much worse than our best case of one.

The approach taken on the ABI for big CHERI is to use an indirection layer, still referred to as a global offset table (GOT), even though it doesn't contain offsets.
This mirrors the baseline RISC-V GOT model, where the same sequence would be:

```
        auipc   a0, %got_pcrel_hi(x)
        lw      a0, %pcrel_lo(.Lpcrel_hi0)(a0)
```

At two instructions, this looks cheap, but note that the second is a load, which is loading the address from the GOT.
This means that, although it's only eight bytes of instructions, it's *also* one pointer's worth of data (eight more bytes for CHERIoT, sixteen for big CHERI).
The second instruction is accessing memory (even on CHERIoT systems with local SRAM, memory-access instructions are typically slower than ALU instructions).

This is worse than our common case, but is often better than our worst case.

The new model
-------------

In the new approach, any global that isn't reachable via a short displacement from the global pointer (i.e. things that would need `auicgp`) is turned into a GOT-relative access.

This also serves as a building block for other optimisations.
The three-instruction sequence is smaller than the GOT-relative access *if a global is accessed only once*, but is larger if the same global has its address taken three or more times.
The linker has full visibility into the number of accesses to a global and so can make this decision.

Similarly, globals that are too big to have their bounds applied in a single instruction will be able to fall back to the GOT mechanism.
These are rare but they do occur for things like frame buffers.
These are close to break even for a single use.
For example, imagine if `x` is an array of 1028 `int`s.
Taking the address is currently this (long!) sequence:

```
	lui             a0, 1
	addi            a0, a0, 16
.LBB0_1:
	ct.auicgp       a1, %cheriot_compartment_hi(x)
	ct.cincoffset   a1, a1, %cheriot_compartment_lo_i(.LBB0_1)
	ct.csetbounds   a0, a1, a0
```

Assuming that the `auicgp` is relaxed away, this becomes:

```
	lui             a0, 1
	addi            a0, a0, 16
	ct.cincoffset   a1, a1, {offset of x}(gp)
	ct.csetbounds   a0, a1, a0
```

This is four instructions, but both the `lui` and `addi` will, in this case, be the 16-bit variants, so this is 12 instructions: smaller than the GOT-relative sequence plus the GOT entry.
This sequence also requires an additional register, and removing that requirement can improve code generation in the rest of the function.

To support this, we've introduced a new `ct.auipcc.data` pseudo-instruction that emits padding nop that is normally relaxed away, but provides space for the linker to emit the longest sequence.
The sequence the compiler will emit looks like this:

```
.LBB4_1
	ct.auipcc.data  s0, %cheriot_compartment_data_hi(x)
	ct.cincoffset   s0, s0, %cheriot_compartment_lo_i(.LBB4_1)
	ct.csetbounds   s0, s0, %cheriot_compartment_size(x)
	ct.cincoffset   a0, s0, a0
```

The linker can then relax this to any of the sequences described above, almost always to something much shorter.

The extra padding is needed only in one specific case, but emitting it in all cases that may use the global pointer simplifies the linker logic.
If the compiler statically knows that the access is in bounds, it will emit the short two-instruction sequence of `auipcc` followed by a load or store.
This is two 32-bit instructions (8 bytes total).
If the global is large, this must be transformed into a GOT load followed by a load or store, which is three instructions, two of which might be 16-bit, but in the worst case all three will need the full range of the 32-bit variants.
The linker always removes the nop but may use the space that it reserved for the longer sequence, in this one corner case.

A final note
------------

CHERIoT benefits enormously from having co-designed the ISA, ABI, and programmer model.
The [`cheriot-audit` tool](https://github.com/CHERIoT-Platform/cheriot-audit) makes it easy to reason about the security of a firmware image.
This tool is possible because we defined our ABI so that everything that a compartment accesses outside of its code and globals region is explicit (in the programmer model).

Several of these optimisations are possible only because we took a holistic view to system design.
If, for example, we allowed globals from other compartments to be transparently accessed then this would have required a GOT approach from the start and would also have made reasoning about communication between compartments much harder.
Instead, we have an explicit notion of a pre-shared object, which must come via a compartment's import table and so is exposed to auditing.
By making the import explicit, we can also go beyond core C features and make the *permissions* explicit, so we can see by reading the source code, and confirm with `cheriot-audit` that exactly what permissions a compartment has on a shared object.

Decades of prior work have shown us that security policies divorced from the source code are hard to reason about and often get out of sync with the implementation.
When they get out of sync, people notice when components have too few permissions (they stop working) but not when they have too many (until an attacker notices).

For CHERIoT, we had an explicit design goal that you should be able to reason about the security of your code by reading the source and do coarser-grained auditing of *other people's code* with additional tooling.
Whether a function is in your compartment or elsewhere is explicit in the source.
Whether a global is uniquely owned by you or a pre-shared object is explicit in the source.
The ABI we co-designed with the programmer model to enable this and it lets us emit very short instruction sequences in the common case, while falling back to larger ones to handle corner cases.
