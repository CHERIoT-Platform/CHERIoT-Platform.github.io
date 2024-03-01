---
layout: post
title:  "Announcing CHERIoT-Audit"
date:   2024-03-01T12:00+00:00
categories: rtos firmware auditing
author: "David Chisnall"
---

For at least two years, I've been talking about how the CHERIoT ABI and compartmentalisation model makes it possible to audit precisely what each compartment is doing.
Possible, unfortunately, is not the same as easy.
When you link a CHERIoT firmware image, you get a JSON report that details what compartments and threads are in the image and how they can communicate.
This can be quite large.
For the RTOS test suite, the JSON report is over 200 KiB (significantly larger than firmware image itself).
This is JSON, and so can be consumed by pretty much any programming language, but ideally there would be an easy way of writing policies over this kind of thing.

Enter: [`cheriot-audit`](https://github.com/CHERIoT-Platform/cheriot-audit).
This tool allows you to write policies in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/), Open Policy Agent's policy language.
Rego is a rich language for writing policies that check some properties over arbitrary JSON documents and so is a perfect fit for our requirements.

This tool requires a few things on the command line:

 - The board description (JSON) file used to build the image.
 - The JSON emitted by the linker.
 - Any Rego modules that you may (optionally) wish to add to provide other reusable policy fragments.
 - A query that will run with access to all of the above.

Let's take a look at some of the things that you can do.
For the rest of this post, we'll assume that you're passing the `test-suite.json` from the test suite (with `-j`) and the `sail.json` that was used to build it for the Sail model (with `-b`).
We'll just provide the Rego query (`-q`) and the output.

The query can be used to inspect various properties.
We can start with some very simple ones.
The simplest queries just show you some fragment from the input JSON.
Let's see what the check-pointer-test compartment exports:

```rego
input.compartments.check_pointer_test.exports
```

```json
[{"export_symbol":"__export_check_pointer_test__Z18test_check_pointerv", "exported":true, "interrupt_status":"enabled", "kind":"Function", "register_arguments":0, "start_offset":1376}]
```

This tells you that it exports a single entry point, which is a function that runs with interrupts disabled.
The symbol name for the export is not very readable if you are a human, so let's use one of `cheriot-audit`'s built-in functions to demangle it:

```rego
export_entry_demangle("check_pointer_test", input.compartments.check_pointer_test.exports[0].export_symbol)
```

```json
"test_check_pointer()"
```

A lot of more interesting properties involve collecting information from different places.
Rego has list and set *comprehensions* for building collections from things that match a particular property.
Let's try extracting all of the allocator capabilities (the static sealed objects that authorise a compartment to allocate some memory):

```rego
[ c | c = input.compartments[_].imports[_] ; data.rtos.is_allocator_capability(c) ]
```

```json
[{"contents":"00040000 00000000 00000000 00000000 00000000 00000000", "kind":"SealedObject", "sealing_type":{"compartment":"alloc", "key":"MallocKey", "provided_by":"build/cheriot/cheriot/release/cheriot.allocator.compartment", "symbol":"__export.sealing_type.alloc.MallocKey"}}, {"contents":"00001000 00000000 00000000 00000000 00000000 00000000", "kind":"SealedObject", "sealing_type":{"compartment":"alloc", "key":"MallocKey", "provided_by":"build/cheriot/cheriot/release/cheriot.allocator.compartment", "symbol":"__export.sealing_type.alloc.MallocKey"}}, {"contents":"00100000 00000000 00000000 00000000 00000000 00000000", "kind":"SealedObject", "sealing_type":{"compartment":"alloc", "key":"MallocKey", "provided_by":"build/cheriot/cheriot/release/cheriot.allocator.compartment", "symbol":"__export.sealing_type.alloc.MallocKey"}}, {"contents":"00100000 00000000 00000000 00000000 00000000 00000000", "kind":"SealedObject", "sealing_type":{"compartment":"alloc", "key":"MallocKey", "provided_by":"build/cheriot/cheriot/release/cheriot.allocator.compartment", "symbol":"__export.sealing_type.alloc.MallocKey"}}, {"contents":"00100000 00000000 00000000 00000000 00000000 00000000", "kind":"SealedObject", "sealing_type":{"compartment":"alloc", "key":"MallocKey", "provided_by":"build/cheriot/cheriot/release/cheriot.allocator.compartment", "symbol":"__export.sealing_type.alloc.MallocKey"}}, {"contents":"00100000 00000000 00000000 00000000 00000000 00000000", "kind":"SealedObject", "sealing_type":{"compartment":"alloc", "key":"MallocKey", "provided_by":"build/cheriot/cheriot/release/cheriot.allocator.compartment", "symbol":"__export.sealing_type.alloc.MallocKey"}}, {"contents":"00100000 00000000 00000000 00000000 00000000 00000000", "kind":"SealedObject", "sealing_type":{"compartment":"alloc", "key":"MallocKey", "provided_by":"build/cheriot/cheriot/release/cheriot.allocator.compartment", "symbol":"__export.sealing_type.alloc.MallocKey"}}]
```

That's correct, but it's not very informative, unless you can read hex.
How about using the function that the `rtos` package exposes to decode those?

```rego
[ data.rtos.decode_allocator_capability(c) | c = input.compartments[_].imports[_] ; data.rtos.is_allocator_capability(c) ]
```

```json
[{"quota":1024}, {"quota":1048576}, {"quota":4096}, {"quota":4096}, {"quota":4096}, {"quota":4096}, {"quota":4096}]
```

That's a lot less raw data and a lot more readable.
It would be nice to know where they came from though, so let's also capture the owning compartment in an object when we construct the comprehension:

```rego
[ { "owner": owner, "capability": data.rtos.decode_allocator_capability(c) } | c = input.compartments[owner].imports[_] ; data.rtos.is_allocator_capability(c) ]
```

```json
[{"capability":{"quota":1024}, "owner":"allocator_test"}, {"capability":{"quota":1048576}, "owner":"allocator_test"}, {"capability":{"quota":4096}, "owner":"eventgroup_test"}, {"capability":{"quota":4096}, "owner":"locks_test"}, {"capability":{"quota":4096}, "owner":"multiwaiter_test"}, {"capability":{"quota":4096}, "owner":"queue_test"}, {"capability":{"quota":4096}, "owner":"thread_pool_test"}]
```

Now we can see exactly which compartment owns which allocator capabilities.

If we want to guarantee that allocation will never fail are a result of interference, we may care more what the sum of all quotas in allocation objects is.
We can do that by tweaking the comprehension to generate an array of the quotas and then using the built-in `sum` function to add them all up:

```rego
sum([ data.rtos.decode_allocator_capability(c).quota | c = input.compartments[_].imports[_] ; data.rtos.is_allocator_capability(c) ])
```

```json
1070080
```

For the test suite, this is a lot because the allocator test compartment intentionally has a capability that allows it to exhaust all memory (it can allocate 1 MiB but typically runs on a system with 256 KiB of RAM).
Now we have the kind of building block that you might use for your policy, let's explore some other things that may be useful in policies.

Which compartments can call the `test_allocator()` function exported from the `allocator_test` compartment?

```rego
data.compartment.compartments_calling_export_matching("allocator_test", `test_allocator\(\)`)
```

Note the back-ticks here: this is Rego syntax for a raw string.
This argument is a regular expression and using normal strings will introduce two levels of escaping, which is confusing.

```json
["test_runner"]
```

That's what we expect: this should be called from the test runner and nothing else.
It would be nice if that property could be simply enforced.
The allow-list predicates make it easy to enforce this kind of thing:

```rego
data.compartment.compartment_call_allow_list("allocator_test", `test_allocator\(\)`, {"test_runner"})
```

```json
true
```

The last parameter to that query was a *set* of compartments that may call this entry point.

Typically, policies will be rules that combine a set of conditions like this.
You can also enforce the same kind of guarantees for memory-mapped I/O regions.
For example, let's make sure that only the scheduler has direct access to the core-local interrupt controller:

```rego
data.compartment.mmio_allow_list("clint", {"scheduler"})
```

```json
true
```

These use functions provided by the `compartment` package, which is part of the cheriot-audit tool.
The name of the device is the name from the board description file.


The `cheriot-audit` tool is still very new and will evolve a lot over the next few months.
Hopefully this provides some hints about the kinds of policy that you may be able to write.
You can then use these policies to drive code-signing decisions.

The JSON reports from the linker also include the pre-linkage hash of every section that ends up in the final image.
If you're incorporating binary-only components from third parties, you can integrate this tooling with SBOMs to ensure that some components in your linked firmware really are the hashes that your supplier promised and to ensure that those compartments do not access anything that you didn't want them to touch.
