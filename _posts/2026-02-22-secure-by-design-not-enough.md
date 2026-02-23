---
layout: post
title: "When 'secure by design' is not enough"
date:   2026-02-22
categories: 
author: Tita Rosemeyer
---


This blog post summarises results from my MSc thesis:

[**"Formal Verification of Capability-Safety in the CHERIoT-Ibex Processor"**, University of Oxford, 2025](https://www.cs.ox.ac.uk/tom.melham/grads/Rosemeyer-2025-FVC)

The full thesis provides a more formal treatment of the methodology, background theory, modelling assumptions, and complete property encodings used in the verification.

---

CHERIoT is built on a powerful idea: if you give software only the authority it needs — and enforce that in hardware — you eliminate entire classes of vulnerabilities. A great deal of thought has gone into making the system secure by design. But designing a secure architecture is only the beginning: 

It still needs to be implemented.

And no matter how carefully thought-through the ideas are, this is where bugs can sneak in. The lowest-level components run with the highest privileges in the system. If something goes wrong there, the entire security guarantee is at risk of collapse. 

We can test and review code. But we cannot manually explore every possible execution path of a privileged component running on a pipelined processor. And when a single missed edge case can invalidate an otherwise sound system, the security provided by manual inspection seems... *fragile*.

So instead of asking “Did we test this thoroughly?”, we can ask a stronger question:

Can we prove that these components never leak sensitive capabilities — under any possible execution?

That is the question that motivated my master's thesis.

## Formal verification, briefly

Formal verification is, at its core, about replacing “we think this is correct” with “we can prove this is correct”.

Testing explores individual executions of a program. You pick inputs, run the system, and observe the result. If something breaks, you have found a bug. If nothing breaks, you gain confidence — but only for the cases you actually exercised.

Formal verification takes a different approach. Instead of sampling executions, it builds a mathematical model of the system and reasons about **all possible executions** at once. You express a property — for example, that a sensitive capability must never be present in the register file at return — as a precise logical formula (in our case, in Linear Temporal Logic). Automated reasoning tools then determine whether there exists any execution that violates this property. If one exists, the tool produces a concrete counterexample. If not, you obtain a proof that the property holds for every possible behaviour of the model.

A proof is only as strong as the model it is built on. But if the model accurately reflects the system of interest, formal verification allows us to move from “this seems safe” to “this property holds for all possible executions”.

In my thesis, I set out to apply this approach to some of the most privileged components in CHERIoT — and to do so at a level much closer to what actually runs on the hardware than is typical in software verification.

## What I verified

I focused on two of the most privileged components in the CHERIoT RTOS: the *unsealer* (part of the allocator) and the *switcher*.

Both sit deep in the trusted computing base. Both manipulate capabilities directly. And both must uphold strict capability-safety guarantees for the entire system to remain sound.

### The unsealer

The unsealer lives inside the allocator, which manages dynamic memory and provides use-after-free protection. In CHERIoT, sealed capabilities are used to construct type-safe, opaque abstractions (see the [introduction to sealed types](https://cheriot.org/sealing/compiler/2025/01/30/introducing-sealed-types.html)).

At the ISA level, sealing combines a pointer-like capability with a sealing key. The result is a capability that can be passed around, but not dereferenced or modified. Unsealing reverses this, but only with the correct authority. As [David explains](https://cheriot.org/rtos/sealing/2025/11/06/sealing.html):

> “The `token_unseal` function unseals the object using the hardware sealing key and then compares the virtual otype in the header to the type of the permit-unseal capability passed as the key. If they match, it returns an unsealed capability to the object without the header, so the caller doesn’t ever have access to the header.”

The `token_unseal` implementation does not rely solely on the user-provided unsealing key. It also accesses a hardware sealing key — effectively a master unsealing capability. That is acceptable because the allocator is trusted, but it is absolutely critical that this authority never leaks. Similarly, the header of the sealed object contains sensitive information that must never be exposed to untrusted code.

The properties I verified were therefore security properties:
- The hardware sealing capability must never be exposed.
- No capability derived from it may remain accessible after execution.
- The header of the sealed object must not leak to the caller.
- The only capability returned must be the correctly unsealed object — and only if the virtual object type matches.



### The switcher
While the allocator controls dynamic memory, the switcher controls execution itself. It is the most privileged component of the RTOS and is responsible for context switching between compartments and threads.

Its responsibilities include:
- Saving register state on interrupts,
- Invoking the scheduler with sealed references to the register save area and trusted stack,
- Restoring general-purpose and capability registers from the trusted stack,
- Installing a new program counter capability (PCC),
- Resuming execution of the selected context.

I focused in particular on the *common context installation procedure* (`.Lcommon_context_install` in [`entry.S`](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/1e41fff9e05e36234e5b1066179abee37730fe91/sdk/core/switcher/entry.S#L1137)), which restores the saved state of the next runnable context and transfers control to it.

The security requirements here are:
- The installed PCC must not retain *Access System Registers* (ASR) permission — otherwise privilege escalation becomes possible.
- All registers — especially the capability stack pointer (CSP) — must be sanitised to prevent cross-context information leakage.

Together, these case studies cover two complementary aspects of capability safety:
- Controlled authority handling in the allocator (unsealing),
- Strict isolation across execution contexts in the switcher.

Both sit at the heart of CHERIoT’s security model — and both operate at the lowest level of the system.

## The key idea - Binary-level verification on RTL

Most formal verification of low-level software operates at the source-code level — typically C or C++. Tools analyse the program and prove that certain properties hold at that level: that a variable is not zero, that a pointer does not escape its bounds, that a particular invariant is maintained.

But in CHERIoT, we care about the actual architectural state of the processor: the contents of the register file, the permissions attached to active capabilities, and whether authority is confined exactly as intended.

This creates an *abstraction gap*. Source code must still pass through compilers, linkers, and runtime layers before it executes. Even verification at the assembly level typically relies on an abstract model of the ISA semantics, not the actual hardware implementation. Compiler bugs, microarchitectural corner cases, or mismatches between specification and silicon remain outside the scope of the proof.

So I took a different approach.

Instead of verifying source code against an abstract model of the ISA, I verified compiled binary code running directly on a cycle-accurate RTL model of the CHERIoT-Ibex processor.

Concretely, I took:
- The compiled binary of the relevant RTOS components,
- The formally verified SystemVerilog RTL model of the processor,
- and treated that combination as the mathematical system to reason about.

This eliminates the compilation gap: the model I reason about is much closer to what actually executes on the device.

It also makes the problem significantly harder.

At the RTL level, there are no variables or functions, only wires. The processor is a transition system: at each clock cycle, signals are either high or low. To prove a security property, I had to express it in terms of these signals across time — in the language of SystemVerilog and temporal logic.

From this, I built a framework that encodes the execution of specific binary instruction sequences and checks the desired capability-safety properties directly against the processor model.

## How this works in practice

At a high level, the problem had three parts:

1. Define what it means for a piece of binary code to execute.  
2. Define the security properties precisely.  
3. Prove that those properties hold for every possible execution.

Each of these required translating informal reasoning about code into precise statements over RTL signals.

### 1. Defining execution in Linear Temporal Logic

The first step was to formalise what it means for a sequence of instructions to execute on the processor.

Since we are working at the RTL level, execution must be expressed in terms of pipeline signals. I chose the **writeback stage** as the “point of truth”. An instruction is considered executed when it reaches writeback without a fetch error. This fixes the final pipeline stage and avoids reasoning about partially executed instructions.

The core macro for identifying an instruction in writeback looked like this:

```verilog
`define INSTR wbexc_decompressed_instr
`define INSTR_WB(instr) \
    wbexc_exists && ~wbexc_fetch_err && `INSTR == instr
```

A straight-line fragment of the `token_unseal` binary:

```nasm
; fragment from the unsealer binary
0: fe4506db ct.cgettag a3, ca0 ; l00_cgettag
4: c281     beqz a3, .Lexit_failure ; l04_beqz
6: fe2506db ct.cgetbase a3, ca0 ; l06_cgetbase
a: 00d51063 bne a0, a3, .Lexit_failure ; l0a_bne
e: fe3506db ct.cgetlen a3, ca0 ; l0e_cgetlen
```

is encoded as a SystemVerilog assertion (SVA) sequence:

```verilog
sequence straight_line_example;
    (`INSTR_WB(l00_cgettag) && instr_will_progress
     ##1 `INSTR_WB(l04_beqz) && instr_will_progress
     ##1 `INSTR_WB(l06_cgetbase) && instr_will_progress
     ##1 `INSTR_WB(l0a_bne) && instr_will_progress
     ##1 `INSTR_WB(l0e_cgetlen) && instr_will_progress);
endsequence;
```

This encoding works naturally for straight-line code. Branching is handled by defining a separate sequence for each feasible path through the program. This was manageable for the relatively structured code in `token_unseal`, but it would require more scalable encoding techniques for larger systems.

Exceptions are treated explicitly: if an instruction can fault, that path must also satisfy the safety condition.

### 2. Formalising capability-safety

Once execution was defined, the next step was to formalise the security properties.

For both the unsealer and the switcher, the core property was:

> No sensitive capability — or any capability derived from it — may remain in the register file after execution.

The “derived from” part is essential. In CHERI, capabilities can be transformed while preserving authority (for example by shrinking bounds or removing permissions). It is therefore not enough to check for equality; we must check for derivability.

This required:

1. A formal definition of what it means for one capability to be derived from another.  
2. A way to scan the entire register file and check for such derivatives.

A typical helper function looked like this:

```verilog
function automatic bit no_derivatives_except_ca0_ca1_fn(reg_cap_t cap, logic [31:0] addr);
    for (int i = 0; i < 32; i++) begin
        if (i != 10 && i != 11) begin
            if (derived_from(`RF.rf_cap[i], regs[i], cap, addr))
                return 0;
        end
    end
    return 1;
endfunction
```

This checks that no register (except explicitly allowed ones) contains a capability derived from a given sensitive capability.

In some cases — especially in the switcher — I also needed to refer to the value of a capability *at entry* to a procedure. To support properties like:

> “The capability in `csp` at entry to common context installation does not remain in the register file upon return,”

I introduced a virtual “snapshot” signal that captures the at-entry value when the first relevant instruction reaches writeback. Later assertions can then compare against this snapshot.

### 3. Assembling and verifying the properties

Execution sequences and safety conditions are combined into full properties of the form:

```verilog
property example_prop;
    (code_sequence and assumptions |-> safety_condition);
endproperty;
```

This reads as:

> If the specified code sequence executes under the given assumptions, then the safety condition must hold at the end.

Branching requires one such property per feasible path. Exception handling is addressed by generating additional properties for prefixes of each sequence:

```verilog
property exception_prop;
    (property_prefix_up_to_line_i
     |-> ~wbexc_err | safety_condition);
endproperty;
```

Informally:

> If an exception occurs after the first *i* instructions, the safety condition must already hold.

These properties are then given to the model checker together with the formal RTL model of the processor. I used **Jasper** by [Cadence](https://www.cadence.com/en_US/home.html):

The proof technique that proved most effective was [**k-induction**](https://www.cs.ox.ac.uk/people/thomas.wahl/Publications/k-induction.pdf), a generalisation of mathematical induction commonly used in hardware model checking. I have my own pet theory on why $k$-induction works well for this problem, but that is a story for another day.


## What I found

### Complete verification of `token_unseal`

For the unsealer, I was able to fully verify the intended capability-safety properties across all normal and exceptional execution paths.

Specifically:

1. The global (hardware) unsealing authority — and any capability derivable from it — is not present in the register file at any exit point from `token_unseal`.

2. The sealed capability appears only in one of two forms:
   - its original sealed form, or  
   - its correctly unsealed form, with the header removed.

In other words, unsealing does not accidentally expand authority, and no privileged capability leaks through subtle execution paths.

### Exploratory results on the switcher

For the switcher, a complete verification was beyond the scope of this thesis. Instead, I focused on a simplified straight-line fragment of the common context installation procedure — the part responsible for restoring state at the end of a context switch.

Under clearly stated assumptions about the execution environment, I proved that the capability stack pointer (`csp`) from the outgoing context is confined exactly as intended during restoration: it is installed in the appropriate system register, and no other register retains authority derived from it. In particular, the fragment does not leak residual capability authority across contexts.

While this is not a full verification of the switcher, it demonstrates that the methodology extends beyond the allocator and can handle privilege transitions and context restoration logic under realistic modelling assumptions.

### A bug uncovered

During verification, one of the exception properties failed.

This exposed a subtle implementation bug: if a sealed capability of length less than 16 bytes was constructed, an exception path could expose the unsealed pointer. The issue was reported and confirmed by the CHERIoT team:

https://github.com/CHERIoT-Platform/cheriot-rtos/issues/550

This was a valuable result proving the usefulness of the approach not only for gaining confidence in the absence of bugs, but also for finding real bugs that could have been missed by testing or code review.

### Scalability and timing

To evaluate scalability, I measured proof time for a representative property across paths of varying length through the unsealer. The results are summarised in the graph below:

![Timing Graph for Branch Properties](image.png)

The x-axis shows the number of cycles in the encoded execution path; the y-axis shows proof time in seconds.

Two observations stand out:
- Proof time scales approximately linearly with path length.
- Even the longest paths tested remained practical to verify (under ~45 minutes per path).

Given that LTL model checking is co-NP complete in general, this near-linear behaviour is encouraging. The combination of structured property decomposition and k-induction appears sufficient to keep verification tractable at this scale.

### Feasibility and next steps

Overall, the project demonstrates that binary-level capability-safety properties can be verified directly against a cycle-accurate RTL model in practice.

Several natural extensions follow:
- **Automation.** Properties and instruction sequences were constructed manually. Automating their generation from binaries would remove the main bottleneck and enable larger-scale verification.
- **Proof optimisation.** Established techniques such as SST tunnelling could further reduce solver cost and make subsystem-scale verification more efficient.
- **Expansion.** The framework could be applied to additional components: for example, extending verification to larger portions of the switcher or other security-critical subsystems.

## Final thoughts

CHERIoT was designed to enforce least authority at the hardware level. This work shows that, at least for carefully scoped components, we can also *prove* that those guarantees are upheld by their implementations — not just by their design.

The results on the unsealer and the exploratory work on the switcher suggest that binary-level verification against a cycle-accurate RTL model is both technically feasible and practically tractable.

There is still work to be done — in automation, scalability, and broader coverage — but the core methodology holds.

For me, the most important takeaway is this: capability safety in CHERIoT does not have to rely solely on trust and testing. It can, in principle, be established by proof.