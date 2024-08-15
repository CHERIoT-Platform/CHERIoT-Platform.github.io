---
layout: post
title:  "Sharing objects between compartments"
date:   2024-08-15
categories: sharing rtos compartments
author: David Chisnall
---

The CHERIoT compartment model is similar to an object-oriented model, where each compartment exposes a set of entry points (analogous to methods) that can be called by other compartments.
This works well for compartmentalising a lot of libraries: expose their public API as compartment entry points.

One of the common questions from people starting to some existing code in a compartment is: How do I export a global from this library?
To which the obvious answer is: what does that even mean?

When you expose a function from a compartment, the security properties are well defined.
Control flow will transition from callers to that entry point.
The switcher will ensure that only things passed as arguments are visible in the callee.
On return, the switcher will ensure nothing except the return value (and things reachable from it) are exposed to the caller.

But what are the security properties when you share a global?
Should every compartment that can access it be able to write to it?
This may be what you want (assuming a small number of compartments can access it).
For example, if you have some performance-monitoring counter where the primary requirement is to minimise the probe effect.
In this case having a compartment write an invalid value is less of a problem than the performance overhead of a cross-compartment call for each update.

In other (more common) cases, you may want to expose an object that one compartment can write to but many can read.
We had one example of this in the core of the RTOS already.
The allocator exposes an epoch counter that it increments when it starts and finishes inspecting a list of hazard pointers (so odd numbers indicate that it's in the middle of a read).
We were (ab)using the mechanism that we have for importing capabilities for memory-mapped I/O regions for this, but it was not a generic mechanism.

Most examples with similar requirements defined a global in one compartment and then exposed an entry point that returned a pointer to it.
For example, the SNTP compartment in the network stack provides [a shared library for getting the current UNIX timestamp using the CPU counter and the last value from NTP](https://github.com/CHERIoT-Platform/network-stack/blob/main/lib/sntp/time-helpers.cc).
A read-only pointer to the value from NTP is fetched by [calling a function exported from the SNTP compartment](https://github.com/CHERIoT-Platform/network-stack/blob/14aa5812109b6e14964c60ebb1cdd08e33af952d/lib/sntp/time-helpers.cc#L19C20-L19C33).
This code would be simpler if it were possible to simply import the cached time as a pre-shared object.

This week, we've added a fully supported abstraction for these use cases.
The first part [introduced the support in the RTOS](https://github.com/CHERIoT-Platform/cheriot-rtos/pull/283)
This introduces macros for [importing a pre-shared object with all permissions](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/96b22d4902a83cb9cce0a84a33ad1d5e4efabdcf/sdk/include/compartment-macros.h#L136) or with [a subset of permissions](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/96b22d4902a83cb9cce0a84a33ad1d5e4efabdcf/sdk/include/compartment-macros.h#L110).
It also extends the build system to allow compartments to [define pre-shared objects that they need](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/96b22d4902a83cb9cce0a84a33ad1d5e4efabdcf/tests/xmake.lua#L89).

Note that the last bit is not the same as defining pre-shared objects that they *export*.
There is no notion of a compartment exporting globals.
Instead, there are pre-shared objects that are imported by one or more compartments.
This distinction is important because there may not be a canonical owner for a global.

When you define a shared object, you specify its name and size.
If two compartments define an object of the same name and different sizes, the build will fail.

With the RTOS bits done, the next part was making pre-shared objects [show up in the linker reports](https://github.com/CHERIoT-Platform/llvm-project/pull/39).
Now, when you define an object, you'll see something like this in the `SharedObjects` section of the linker report:

```json
{ 
  "end": 2147605688,
  "name": "exampleK",
  "start": 2147604664
}
```

This describes the start and end address of the object and its name.
In this case, it's a 1 KiB object called `exampleK`.
You'll also see a corresponding entry in the `imports` section for anything that imports this object, for example:

```json
{
  "kind": "SharedObject",
  "length": 1024,
  "permits_load": true,
  "permits_load_mutable": true,
  "permits_load_store_capabilities": true,
  "permits_store": true,
  "shared_object": "exampleK",
  "start": 2147604664
}
```

This shows the object name, its address and length (which may be smaller than the global in the future, though always match it for now).
It also defines the set of permissions that this has.

As with the rest of the linker report, we don't expect normal humans to ever read this directly.
This brings me to the last part, the [cheriot-audit integration](https://github.com/CHERIoT-Platform/cheriot-audit/pull/7).

This adds some helper functions for inspecting shared objects.
For example, we have two pre-shared objects associated with the allocator.
The hazard-pointer list is accessible only by the allocator (a subset of it for the current thread can be read via a call to the switcher).
The epoch counter can be read by anything but must be written only by the allocator.
We have added this to the RTOS policy like this:

```rego
data.compartment.shared_object_allow_list("allocator_hazard_pointers", {"allocator"})
data.compartment.shared_object_writeable_allow_list("allocator_epoch", {"allocator"})
```

If the `allocator_hazard_pointers` object is accessible by any other compartment or if `allocator_epoch` is writeable by anything except the allocator, this will fail.
For some defence in depth, we also restrict the permissions with which the allocator imports the hazard pointer array and so we can also check that we got that right in the auditing policy:

```rego
some hazardListImport
hazardListImport = [ i | i = input.compartments.allocator.imports[_] ; i.shared_object == "allocator_hazard_pointers"]
every i in hazardListImport {
    i.permits_load == true
    i.permits_load_store_capabilities == true
    i.permits_load_mutable == false
    i.permits_store == false
}
```

The first two lines use a [Rego comprehension](https://www.openpolicyagent.org/docs/latest/policy-language/#comprehensions) to collect every import from the allocator compartment that refers to the hazard pointers object.
We then assert that, for every one of those imports, the permissions are the same and permit loading capabilities, but not storing them or storing through any loaded capabilities.

This kind of policy is easy to write and flexible.
As with other CHERIoT policies, it's up to you how you use them.
You can use this to drive the code-signing choices for built firmware, make your build fail entirely if they fail, or just use them for introspection.
