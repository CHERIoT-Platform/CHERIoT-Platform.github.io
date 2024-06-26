---
layout: post
title:  "Sentries for control-flow integrity"
date:   2024-06-26
categories: ISA Ibex
author: "David Chisnall"
---

For a long time, CHERI platforms have had a notion of a *sealed entry* (sentry) capability.
These use the sealing mechanism (that makes a capability unusable and immutable unless you unseal it with another capability that authorises the unsealing) to provide capabilities that you can jump to, but not otherwise modify.
Jumping, from the CPU's perspective, means installing a new value in the program counter register.
On a CHERI system with sentries, that operation also unseals the capability.
This means that the caller can't jump to the middle of a function and can't access any data that the function would access via its program counter (capability).

CHERIoT extended this with sentries that enable and disable interrupts.
This ensures that interrupt disabling follows a structured-programming pattern.
You can call a function that runs with interrupts disabled and the original interrupt state is captured in the link register.
If you're running with interrupts enabled, call a function with interrupts disabled, then return, the return will automatically restore the interrupt-enabled state.

This lets you reason about worst-case execution timing conveniently and, using [cheriot-audit](https://github.com/CHERIoT-Platform/cheriot-audit), write policies that allow only a small allow list of interrupts-disabled functions to be called.
Our atomics library provides interrupts-disabled read-modify-write functions for use on cores that don't natively support atomics.
You can imagine also providing a library that does things like inserting entries into a linked list, with guaranteed worst-case execution time, and allowing those to be called from any compartment.
Being able to call these functions does not allow a compartment to disable interrupts for an unbounded amount of time, only for the duration of the calls.
You can separately audit these functions for worst-case execution time and then reason about how long a compartment that can call them can run with interrupts disabled.

We realised a little while ago that there was a possible attack:
Imagine that you have an interrupt-disabled sentry (for example, one of the atomic increment functions from the atomics library).
You can call this normally and return, and everything is fine.
If; however, you put that sentry in the return register (`cra`) and then jump to the sentry with a jump (rather than jump-and-link) instruction, the callee will return by jumping to its own entry point.
It will then run again, and return to itself.
This is an infinite loop that runs with interrupts disabled.

This doesn't matter for a lot of cases because the attacker must be able to provide arbitrary instructions to run.
If you control all of the code on the system and audit any third-party assembly, this kind of thing won't be possible.
If you want to be able to take binary-only compartments from third parties and integrate them with your firmware image, secure in the knowledge that your policy restricts the damage that a supply-chain attack can do, it's more of an issue.
They can't (with this attack) compromise confidentiality or integrity, but they can make your device enter a state where a hard reset is required.

This kind of attack could be prevented if we had a way of differentiating between forward-edge (function call) and backward-edge (return) sentries.
We'd always planned on doing that in CHERIoT v2 but we'd held off because we thought that it would require us to introduce new instructions.
RISC-V does not have a dedicated return instruction but backwards sentries should be used only with returns and so we needed a way of architecturally expressing the intent that something was a call or a return.

Last month, Murali Vijayaraghavan [proposed an extension that would allow these to be introduced in a backwards-compatible way](https://github.com/microsoft/cheriot-sail/pull/54).
This works by treating the instruction differently depending on the register operands.
RISC-V has a single instruction for both jump to a register and jump-and-link to a register value.
The non-linking version just uses the zero register (`cnull` on a CHERI system) as the link register.
Murali's proposal ended up with the following interpretations of the instruction depending on the operands:

cs1    | cd               | Used for        | Valid sentry types
--     | --               | --              | --
`cra`  | `cnull`          | Function return | Return sentries
≠`cra` | `cnull`          | Tail call       | Unsealed or interrupt inheriting forward sentry
any    | ∉{`cnull`,`cra`} | Function call   | Unsealed or interrupt inheriting forward sentry
any    | `cra`            | Function call   | Unsealed or forward sentries

We are treating `cjr cra` as a return instruction, so it must have a return.

We can use `cjr` with any register *except* `cra` for tail calls or for jump tables.
The compiler already avoided `cra` for these cases because it needed to pass the return address in (for tail calls) or keep it (to return to later).

Any `cjalr` (i.e. the variant that uses a link) can be used for function calls, but we require `cra` to be used as the link register when using a sentry that changes the interrupt state.
This ensures that any function that changes the interrupt state knows the register that it will return to is provided by the branch.
This precludes tail-call optimisation of functions that change interrupt state, but that's rarely useful (interrupt-state-changing functions are rare anyway).

We ran a number of tests with the Sail model and found that almost everything worked.
The one exception was [a trick that we were doing in the start of the loader](https://github.com/microsoft/cheriot-rtos/commit/3bb8a1ba66c1a1c4ae6e501f4529a88dd23076ed) to reuse some code both as a subroutine an as a trampoline for invoking the C entry point.
This wasn't actually needed and so we were able to remove it.

This new behaviour was then [added to the CHERIoT Ibex core](https://github.com/microsoft/cheriot-ibex/commit/da008de45728aab0aac949f5b9c4307bbe7913a1).
We test the Verilator simulation of the CHERIoT Ibex in CI on every RTOS comment and so we were confident that everything was fine.

Then, yesterday, we started getting reports that [CHERIoT RTOS firmware no longer worked on the Arty A7 (FPGA)](https://github.com/microsoft/cheriot-rtos/issues/261).

On the Arty A7, we don't have a working JTAG loader and so we have a small ROM loader that can read the firmware over the UART.
This was working, so we hadn't broken the UART or anything obvious in the core, but we got no output from the loaded firmware at all.
It turned out that, after loading the firmware image, the ROM loader put the address of the entry point in `cra` and then jumped to it with `cjr cra`.
This, of course, no longer worked: the value in `cra` was not a return sentry.

The fix was very simple (just use a different register), but it showed that CFI was working correctly: trying to return to something that was not a valid return address trapped!

This meant that we now had a working (coarse-grained) CFI scheme.
It's already difficult to attack a CHERIoT system, because the first step in most control-flow hijacking attacks is some kind of memory-safety bug and we mitigate those.
It may be possible to copy return addresses via a stack uninitialised-use bug and then place those in a structure that contains a function pointer, but this will now trap.
Similarly, the kind of active attack on availability from a supply-chain attacker (who can inject arbitrary code as long as it passes your firmware policy) is also not possible.

