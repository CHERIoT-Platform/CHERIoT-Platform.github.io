---
layout: post
title:  "Introducing sealed types"
date:   2025-01-30
categories: sealing compiler
author: David Chisnall
---

Sealing is one of the most important parts of CHERI because it enables *usable* compartmentalised interfaces.
Sealing lets you build *type-safe* opaque types that are safe in the presence of mutual distrust and delegation.
In the most recent updates to the compiler and RTOS, we've made this even more friendly for programmers.

# Opaque types are handles

It's very common for a piece of code to hand out some kind of opaque value that can be passed back.
Within a trust domain, a lot of libraries expose *opaque types*.
For example, in C, your header might look like this:

```c
struct PrivateType;

PrivateType *create();
void some_operation(PrivateType*);
```

Callers don't know what the contents of a `PrivateType` are, they just pass around pointers as opaque handles.
This is purely a software-engineering boundary not a security one.
A caller could cast this to a `char*` and overwrite it with arbitrary data, or learn the internal type and read the contents.

At the system call boundary, most operating systems have a similar abstraction, for example `HANDLE`s on Windows or file descriptors on POSIX systems.
The kernel maintains a table of objects for each process and the file descriptor or equivalent provides a key for looking up objects in this table.
You can't use another process's file descriptors because your file descriptor number is an index into *your* table.

The file-descriptor model works well when you have just one kernel (which manages all handles) and userspace processes are a single trust domain.
You can maintain a per-process table and not worry that one part of a process can forge the file descriptor numbers.
When you start decomposing a monolithic kernel, managing access rights like this becomes one of the more complex parts of the system.
CHERIoT, as a hardware-software co-design project aimed to dramatically reduce this software complexity via some useful hardware abstractions.

# Using sealing for opaque types

In CHERI systems, you can *seal* a CHERI capability (a pointer) and then it becomes an opaque token.
CHERI capabilities contain an object type field.
When you seal a capability, this field stores a value taken from the capability used to seal them, which in turn is used to determine which capabilities may unseal it.

You can store a sealed capability in memory, copy it around, but you can't use it.
The only operation that you can do with a sealed capability is unseal it, and you can do that only with the correct capability that authorises unsealing *this specific kind of* sealed capability.
In C, sealing might look something like this:

```c
char *ptr = strdup("hello world");
char *sealed = cheri_seal(ptr, someKey);
// Don't do this, it will trap:
// sealed[0] = 'H';
// This will not trap, but it will clear the tag:
// sealed = sealed + 1;
// Unsealing with the correct key gives the original
char *unsealed = cheri_unseal(sealed, someKey);
assert(cheri_is_equal_exact(ptr, unsealed));
// Unsealling with the wrong key gives an untagged value
char *untagged = cheri_unseal(sealed, wrongKey);
assert(cheri_tag_get(untagged) == 0);
```

Sealed pointers are subject to the same temporal safety mechanisms as any other allocation on CHERIoT and so implementing safe opaque pointers with sealing is easy.
You create a sealed allocation and simply pass the sealed capability out of your compartment.
When you receive one, unseal it and if the unseal succeeds it is a valid value of the type that you expect.
When you free the object, all pointers to it (sealed or unsealed) become untagged and so all outstanding handles are automatically invalidated.

In CHERIoT RTOS, we used `SObj*` as a placeholder for all sealed types.
This meant that we got compile errors if you tried to use them in most invalid ways but it also meant that you couldn't differentiate between two sealed types, such as an allocator capability and one that authorises access to a message queue.
This was annoying because some APIs needed to take two different sealed types as arguments.
It was also somewhat annoying because sealing enforces type safety, but you needed to explicitly specify the type that you were trying to unseal an object as.

# Sealed pointers in the type system

[PR 109](https://github.com/CHERIoT-Platform/llvm-project/pull/109) in CHERIoT LLVM introduced `__sealed_capability` as a type qualifier.
Attempting to dereference a sealed capability or modify its address is an error and will now fail at compile time, rather than run time.
This includes modifications via the CHERI builtins.

The seal builtin now returns a `__sealed_capability`-qualified pointer and the unseal builtin requires one.
This means that, rather than making every sealed pointer a `SObj*`, we can make an allocator capability a `struct AllocatorCapabilityState *__sealed_capability)` and, when we unseal it, get back a `struct AllocatorCapabilityState *`.

This makes safely exposing opaque types from compartments even easier than it was before.
The C macros and C++ inline functions that wrap the builtins preserve the types, so if someone hands you a `T*__sealed_capability`, unsealing it will give you a `T*` that is either a valid `T*` or untagged.

The compiler allows sealed pointers to be implicitly cast to `void*`, to preserve `void*`'s anything-pointer semantics.
This was important for C++, where we would have otherwise needed `void*__sealed_capability` overloads in a lot of places.
Casting back requires an explicit cast, as does casting to any other unsealed type.

# Backwards compatibility

As previously mentioned, all of our existing code used `SObj*`, so adopting this required some changes in the RTOS ([PR 427](https://github.com/CHERIoT-Platform/cheriot-rtos/pull/427)).
We have added a `CHERI_SEALED(T)` macro that takes a pointer type and expands to `SObj*` for the old compiler or `T __sealed_capability` for the new compiler.

If you have existing code that depends on `SObj*`, you can define `CHERIOT_NO_SEALED_POINTERS`, which will revert to the old behaviour.

This change is now also reflected in the `CHERI::Capability` wrapper template, which now takes a second argument to indicate whether the pointer it owns is sealed.

# C++ helpers

Exposing sealing into the type system also means that you can use it in type operations in C++.
We have added a `CHERI::remove_sealed` / `CHERI::remove_sealed_t` template that takes a sealed pointer type as an argument and removes the sealing qualifier.
Similarly, we have a `CHERI::is_sealed_capability` / `CHERI::is_sealed_capability_v` predicate template that allows you to query whether a capability is sealed.
This composes with C++ Concepts to allow you to define overloads that require sealed or unsealed pointers.
