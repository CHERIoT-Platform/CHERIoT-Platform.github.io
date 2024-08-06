---
layout: post
title:  "How to talk to your parents about hardware memory safety"
date:   2024-08-06
categories: cheri
author: David Chisnall
---

Some conversations are difficult to have with members of older generations who grew up with different social norms.
In particular, when you're talking to people who grew up with PDP-11s with their completely flat memory, or Lisp machines or Burroughs Large Systems with their deeply opinionated and language-integrated hardware memory safety, you may find it hard to talk about CHERI.
This guide aims to help you have those conversations with the minimum of stress on both sides.

## You don't understand me or my object model!

When we talk informally about CHERI, it's tempting to say things like 'CHERI provides memory safety' or 'CHERI gives you control-flow integrity'.
The CHERI project started 14 years ago and people who have been working on it for a decade or so know that this is a shorthand but when you're engaging with people who haven't yet accepted the truth of capability systems into their life, it's important to be precise.

A CPU instruction set architecture (ISA), or an ISA extension, for general-purpose computing, cannot provide memory safety.
A definition of memory safety starts from a definition of an object model.
C has an object model, as has Rust, Java, and so on.
Each object model defines the bounds of objects and their lifetime.
If you access an object after the end of its lifetime, or outside of its bounds, you have violated memory safety.
A CPU can't provide memory safety because it doesn't know what the object model for the running program is.

The same is true for control-flow integrity (CFI).
CFI defines a program as a directed graph of blocks of instructions with arcs in between them representing things like function calls, returns, and so on.
Again, the existence of this graph is a property of the language.
For example, in C++ if you call a virtual function then the object on which it's called must be a subclass of the class on which the function is defined.
This set of properties can be quite rich and so most CFI schemes focus on preventing particularly dangerous invalid transitions, rather than preventing *all* invalid transitions.
For example, a C CFI scheme may allow you to call `system` when you meant `puts`, because they take the same argument type, but prevent you from calling `fputs`.
This is a dangerous conflation (it may allow an attacker to run a program rather than printing a string) but it doesn't corrupt the state of the running program.


## You're in my language and you'll follow my language's abstract machine's rules!

So what's the point of CHERI if an ISA can't give you these properties?
CHERI is not designed to give you an object model that you must conform to, it's designed to *give language implementers tools to enforce these properties*.
This is very different from a lot of earlier memory-safe system.
Intel's [iAPχ 432](https://en.wikipedia.org/wiki/Intel_iAPX_432) was designed around the Algol model (though not very well), as was the [B5500](https://en.wikipedia.org/wiki/Burroughs_Large_Systems#B5500) (somewhat better).
Various [Lisp machines](https://en.wikipedia.org/wiki/Lisp_machine) implemented the Lisp memory model.
Such an approach is not feasible today, in a world where most programs use components written in a variety of languages.


That's a very important distinction and it becomes even more important when you ask the follow-on question: Against whom, exactly, are these properties being enforced?

We assume that a compiler has some notion of an object model.
It may be able to enforce that memory model entirely because the source language has strong typing guarantees.
For example, we have a JavaScript VM in the CHERIoT RTOS repository that builds its own garbage-collected type-safe object model using 16-bit integers to represent object pointers (more on that later).
That compiler needs to generate binaries that interoperate with other code.
The other code may be third-party code compiled with the same compiler but [rely on exploiting compiler bugs to bypass certain guarantees](https://github.com/Speykious/cve-rs).
It may be written in a different language with different guarantees, or it may be assembly code with no enforced concept of types or an object model.
Our goal is to allow the compiler for one language to enforce these properties against all of that other code, irrespective of the source language or compiler (if any) used to produce the other code.

Note that this is not just about providing *C* memory safety.
A lot of the work so far is on C and C++, but the same problems occur in safer languages such as Java or Rust.
For example, here's a snippet from the Rust FFI manual that calls a C function from the `snappy` library:

```rust
use libc::size_t;

#[link(name = "snappy")]
extern {
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
}

fn main() {
    let x = unsafe { snappy_max_compressed_length(100) };
    println!("max compressed length of a 100 byte buffer: {}", x);
}
```

Note that the call from Rust to C requires the `unsafe` keyword.
This recognises the fact that Rust enforces a lot of useful properties, such as lifetime safety, that C does not.
As soon as you transition into C code, these properties are no longer automatically enforced.
As a Rust programmer, you are expected to enforce these properties at the API boundary, but in the general case that is impossible on most hardware.
For example, there's no Rust wrapper that you can write that prevents a C library (on non-CHERI hardware) from mutating an immutable object, or from capturing a borrowed reference.

But what if the hardware could help?
What if the Rust compiler could tell some combination of the hardware and the language-agnostic run-time system that a pointer must not be captured, or may be used with load instructions but not stores?

## So what is this chariot thing you kids are talking about?

CHERIoT is a hardware-software platform built on a variant of CHERI that is optimised for small embedded and IoT applications.
It builds on all of the prior CHERI research and makes a set of design choices optimised for tiny low-power devices.

CHERIoT allows a compiler to enforce a fairly rich set of properties against all of the other parts of a program.
It can guarantee that, when invoking untrusted code, the untrusted code cannot:

 - Access objects unless passed pointers to them.
 - Access the memory that was formerly used for an object after that object has been freed.
 - Hold a pointer to an object with automatic storage duration (an 'on-stack' object) after the end of the call in which it was created.
 - Hold a temporarily delegated pointer beyond a single call.
 - Modify an object passed via immutable reference.
 - Modify any object reachable from an object that is passed as a deeply immutable reference.
 - Tamper with (or access) an object passed via opaque reference.

Some of these are baseline guarantees from CHERI, some are built atop those with additional hardware extensions, and others are built atop them with some language-agnostic software.
Let's go through them one at a time.

### It's rude to point!

Running code cannot access objects unless passed pointers to them.
This is a basic property of CHERI but also the absolute minimum foundation for *any* form of memory safety.
If malicious or buggy code can materialise (or forge) arbitrary pointers out of thin air and use them, it can bypass any of the other rules.
In any CHERI system, any instruction that accesses memory (loads, stores, jumps, and so on) must take a capability as one of its operands.
A capability (in general) is an unforgeable token of authority that allows you to do some action.
A *capability system* is one in which all operations require an authorising capability to be presented.
A CHERI capability is a machine word that is interpreted, and protected, by the CPU and which can be used as a hardware type that *implements the language notion of a pointer*.
CHERI capabilities were designed to allow everything that correct C does with pointers, which is a superset of what most safer languages permit.

That capability is protected both in registers and memory by a tag bit.
The tag bit is an attestation from the hardware that there is a valid chain of provenance from the root capabilities that authorise everything down via a sequence of subsetting operations that end up with the capability that you hold.
On CHERIoT, for example, if you have a capability to a heap object, this is telling you that the allocator gave out a capability to a subset of the heap (which itself was given a subset of the address space by the loader).
The CPU doesn't know any of the bits about the software model, of course, it just ensures that you didn't create a fake valid pointer.
The software model defines what valid paths exist between the initial boot state and normal execution with you holding a pointer to a heap object, the hardware guarantees that some such path must have existed for you to hold that pointer.

### I said you could look, not touch!

The two guarantees about immutability (code may not modify an object passed via immutable reference or any object reachable from a deeply immutable reference) are simple properties once you have unforgeable CHERI capabilities.
These are both enforced via *permissions*.
Each capability carries a set of permissions that defines the set of things that it can be used for.
Removing store permission means that it can't be used with store instructions (and so gives a read-only pointer).

Deep immutability isn't just about controlling to top level pointer, you also need to make sure no pointer loaded by following pointers from the original object is ever used to store.
Morello and CHERIoT also define a load-mutable permission.
This permission allows you to transitively load capabilities that have store permission.
When you load a capability, its store and load-mutable permissions are anded with load-mutable permission of the capability that authorised the load.
This gives a very simple way of enforcing deep immutability.
Removing a permission (and so constructing a deeply or shallowly immutable reference) is a simple register-register operation, less expensive in hardware than addition.

Permissions and bounds are monotonic.
You can remove permissions from a capability but not add them.
You can shrink the bounds, but not increase them.
This means, for example, that one function can be given a pointer to a structure and can then create a read-only pointer to a single field of that structure to pass to another function.

You don't necessarily need your source language to map this into the type system.
A language that has a notion of read-only views of objects could use it automatically but in C/C++ we expose the operations to remove (and check) permissions as built-in functions, so you can use them for building your own security policies.

### Why is there a seal living in my computer?

The ability to hand out tamper-proof opaque references is implemented with CHERI's *sealing* mechanism.
CHERI sealing associates an object type (a numerical value) with a pointer and, at the same time, makes the pointer unusable.
Sealing a capability requires a second capability that authorises sealing with the specific type.
Unsealing similarly requires a capability that authorises unsealing with a specific type.
You can use this to enforce *type safety* for opaque types in the presence of unsafe code.

I used this extensively in the [CHERI-JNI](https://www.cl.cam.ac.uk/research/security/ctsrd/pdfs/201704-asplos-cherijni.pdf) work back in 2017.
When a Java program passed an object pointer to C, it was passed as a sealed capability.
C code could do nothing with this other than pass it back to the Java VM via Java Native Interface (JNI) functions.
The JNI exposes functions for getting or setting fields and calling methods.
Each of these could unseal the object and know that it was a valid Java object.
From there, there was no need for CHERI to be involved with type safety because every Java object carries a pointer to its class and so type checks were possible to implement purely in software.
This highlights one of the core goals of CHERI: It *enables* languages to be safe, it doesn't *mandate* how they enforce that safety.
The Java VM can efficiently enforce type safety (and therefore memory safety) within its own world without CHERI (though CHERI can improve performance in a few places), but CHERI enables it to retain these properties even when calling C.

As with permissions, you don't need these to be used directly in the language.
In CHERIoT, we use sealed objects for almost anything where one compartment wants to provide a handle that lets other compartments ask it to do something.
This includes allocating memory, reading or writing message queues, connecting to network servers, and so on.
Cross-compartment type safety is useful even when you don't have a type-safe language.

### It's a free memory!

The remaining three properties are somewhat more specific to CHERIoT.
Preventing an object from being accessed after it has been freed is possible on other CHERI systems but is implemented in different ways.
[Cornucopia Reloaded](https://www.cl.cam.ac.uk/research/security/ctsrd/pdfs/202404asplos-cornucopia-reloaded.pdf) explains an efficient way of implementing it on very large CHERI systems.
CHERIoT takes an approach tailored for tiny embedded systems with no MMU.
Each granule (8 bytes, by default) of memory that can be used for heap (which may not be all memory) has an associated bit in a bitmap.
When an object is freed, the allocator sets all of the associated bits, marking the memory as freed.
From this point on, you can never load a pointer to that object.
Such pointers may continue to exist in memory but the CPU will clear their tag bit when you attempt to load them.
This catches use-after-free errors immediately but is not sufficient for the program to be able to safely reuse the memory.

Eventually, the *revoker* (part of the hardware on CHERIoT) deletes pointers to freed objects from memory.
After that has happened, the allocator can clear the bits in the 'freed' bitmap again and allow the memory to be reused.
The monotonicity properties of CHERI mean that a pointer that points to an object can be used only to derive objects with equal or smaller bounds.

In CHERIoT, the memory allocator itself is just a special case of a language runtime.
It hands out pointers to objects, marks them as free, and periodically triggers revocation, to ensure that no other component (irrespective of the language it's written in) can access them after they've been freed.

This can be used as a foundation for other allocators.
The [Microvium](https://microvium.com) JavaScript VM runs on CHERIoT and provides a garbage-collected heap for JavaScript.
The Microvium heap is a modified semi-space compacting collector.
Each object pointer in JavaScript is a 16-bit integer giving an offset into a logical heap space, which is actually constructed from a set of chunks.
When the garbage collector is run, it will scan the chunks to find live objects, then allocate space to copy them into, and free the old chunks.

This means that every time C code captures a pointer to something on the JavaScript heap (for example, a string returned from `mvm_toStringUtf8`), that pointer will always point to the same object until the GC runs.
After the GC runs, the pointer will become invalid.
C code can use the `mvm_Handle` abstraction to keep objects live across collections, but (on CHERIoT platforms) if it doesn't then it will have an invalid (unusable) pointer, not a pointer to a different object in the JavaScript heap.

Microvium also marks pointers to strings that it hands to C as read-only (no store permission), so JavaScript can do zero-copy sharing with C, but C code cannot violate the type-safety properties of JavaScript.

The final two properties are both variations of the same thing.
It's possible to pass a pointer to a function and ensure that the function does not capture that pointer.
This is implemented with two properties on CHERIoT.
The first is the two-bit information-flow-control model that CHERI has had since the very early days.
This defines some capabilities as 'global' (and, conversely, the rest as local) and adds a store-local permission.
You may store local capabilities only via a capability that has store-local permission.
Global is not a permission (it doesn't authorise you to do anything) but behaves like one: it can be cleared, but not set.

On a CHERIoT platform, the only memory that you will ever see with the store-local permission is the stack.
This means that anything that you remove the global bit from can be held in registers or stored on the stack, but not stored on the heap or in globals.
CHERIoT also adds a permit-load-global permission, so you can make this a deep property: no pointer loaded (at any depth of indirection) from this pointer may ever be stored on the globals or heap.
This is combined with stack clearing in both directions of cross-compartment calls (with a bit of hardware assistance) to implement shallow or deep no-capture guarantees.
A Rust compiler, for example, could use this to pass borrowed references to a C function and have a strong guarantee that the C code could not access the object beyond the function's return.

## So you mean compilers and ISAs can be friends?

None of these language-level properties come exclusively from the hardware.
The hardware provides tools that the compiler can choose to use.
The compiler can also choose not to use them when it doesn't need to.
For safe languages, a lot of properties are guaranteed by construction and so don't need enforcing *within that language's code* (or within the safe subset, for a language like Rust), though a compiler for safety-critical systems may choose to use them for defence in depth against compiler bugs.
In C, for example, we don't enforce stack temporal safety within a compartment because it's easy for static analysers to track this kind of bug when they can see all of the code and it's a better security-performance tradeoff to recommend that people aim the gun slightly away from their foot.
We do enforce it at compartment boundaries, because we're crossing a security boundary.
If you choose not to run a static analyser that checks for stack temporal safety issues, you can introduce bugs into your own code, but not other compartments.

The same applies for CFI properties.
CHERIoT provides a trusted stack for cross-compartment calls and a switcher that enforces a lot of properties on both the call and return path.
Within a compartment, we provide forward and backwards return sentries for coarse-grained CFI (you can't confuse function pointers and return addresses).
This provides a few guarantees at library boundaries.
You cannot jump into the middle of a library function (easily, you may be able to if it spills its program counter to the stack and doesn't zero it on return).
You cannot call a library function that disables interrupts without using the link register to return.
Compilers can build richer abstractions for CFI, or they can accept that memory safety plus some coarse CFI still makes life very hard for attackers.
This choice is not made for them and, if attackers find clever techniques for starting code-reuse attacks in the future, compilers can add additional defences within a compartment.

All of this is why it's important not to talk about properties that CHERI systems enforce in terms of language-level properties.
CHERI provides tools that allow implementations to enforce language-level properties but it does not define any of these in terms of language-level constructs.
This is particularly apparent when, for example, you create a deeply immutable pointer in C code: something that you cannot enforce with language semantics (C lets you cast away `const`) but which CHERI can enforce even on assembly code that handles that pointer.
CHERI doesn't give C memory safety, CHERI gives you a set of tools that allow C, C++, Rust, Java, and so on to all share objects without letting any of them violate the safety properties of the others.
