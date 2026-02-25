---
layout: post
title:  "Post-Quantum Cryptography on CHERIoT"
date:   2025-12-12
categories: pqc
author: David Chisnall
---

When you tell everyone you're building a secure platform, the first thing that they ask about is encryption.
And, in 2025, the hot topic in encryption is algorithms that are safe from hypothetical quantum computers that, unlike real ones, can factorise numbers bigger than 31.
These algorithms are referred to as post-quantum cryptography (PQC).
Since NIST standardised a few such algorithms, there's been a lot more interest in seeing them in production, so I spent some time getting the implementations from the Linux Foundation's PQ Code Package to run on CHERIoT.
A lot of companies are building hardware to accelerate these operations, so it seemed useful to have a performance baseline on the CHERIoT Ibex, as well as something that can be used in future CHERIoT-based products.

What are ML-KEM and ML-DSA for?
-------------------------------

I am not a mathematician and so I'm not going to try to explain how these algorithms work, but I am going to explain what they're *for*.

Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM) is, as the name suggests, an algorithm for key encapsulation.
One side holds a public key and uses it (plus some entropy source) to generate a secret in both plain and encapsulated forms.
The encapsulated secret can be sent to a remote party who holds the corresponding private key.
The receiver can then recover unencrypted version of the secret (and detect tampering).
Now, both parties have the same secret and can use it with some key-derivation function to produce something like an AES key for future communication.

Note that this is somewhat more restrictive than traditional key-exchange protocols.
You don't get to exchange an arbitrary value, the generation step is part of encapsulation.
This also means that it's a fixed size, defined by the algorithm, which is why you typically feed it into a key-derivation function rather than using it directly.

Module-Lattice Digital Signature Algorithm (ML-DSA) has a similarly informative name.
It is intended for providing and validating digital signatures.
It takes a private key, an arbitrary-sized document and context, and produces a signature.
A holder of the associated public key can then validate that the document matches the version signed with the private key and context.

These are both quite low-level building blocks for higher-level protocols.
For example, TLS can use ML-KEM for key exchange and ML-DSA for certificate validation, but also incorporates traditional algorithms in case the PQC algorithms have unexpected weaknesses against classical computers.

Initial porting
---------------

As is usually the case for CHERIoT, porting the C implementations of ML-KEM and ML-DSA required no code changes.
I worked with upstream to slightly simplify the platform-integration layer, so we just provide a single header describing the port.
For example, the [port header for ML-DSA](https://github.com/CHERIoT-Platform/cheriot-pqc/blob/main/include/mldsa_native_config.h) configures the build to produce ML-DSA44 support, defines a custom function for zeroing memory and getting entropy, and adds the `__cheriot_libcall` attribute to the all exported APIs (so we can build them as shared libraries, rather than embedded in a single compartment).
The [file for ML-KEM](https://github.com/CHERIoT-Platform/cheriot-pqc/blob/main/include/mlkem_native_config.h) is almost identical.

With these defined, it is possible to build both libraries as CHERIoT shared libraries.
This motivated a bit of cleanup.
We have a device interface for entropy sources, but it wasn't implemented on the Sail model (which doesn't have an entropy source).
It has a way of exposing the fact that entropy is insecure, so that wasn't a problem, it just needed doing, so I refactored all of the insecure entropy-source drivers to use a common base.
Most encryption algorithms want an API that fills a buffer with entropy.
It's nice if these don't all need to touch the driver directly, so I created a compartment that provides this API and exposes it.
Now, both libraries are simply consumers of this API.
This also makes it easier to add stateful whitening for entropy drivers for hardware entropy sources that don't do the whitening in hardware.

Most CHERIoT stacks are on the order of 1-2 KiBs.
The PQC algorithms use much more space.
More, in fact, than we permitted.

The previous limitation was based on the precision of bounds rounding.
A CHERI capability compresses the bounds representation by taking advantage of the fact that, for a pointer to an allocation, there is a lot of redundancy between the address of the pointer, the address of the end of the allocation (the top), and the address of the start of the allocation (the base).
The distance from the address to base and top are stored as floating-point values with a shared exponent.
In practical terms, this means that the larger an allocation is, the more strongly aligned its start and end addresses must be.
The same restrictions apply for any capability that grants access to less than an entire object.

When you call a function in another compartment, the switcher will truncate the stack capability so that the callee sees only the bit of the stack that you weren't using.
The top and base of the stack must be 16-byte aligned (as an ABI requirement), but a very large stack may have hardware requirements for greater alignment and so may require a gap between the bottom of the caller's stack and the top of the callee's.

Fortunately, we'd added an instruction precisely for this kind of use case: `CSetBoundsRoundDown`.
This takes a capability and a length and truncates it to *at most* that length.
It was a fairly small tweak to the switcher to make it do this, and a much larger amount of time with SMT solvers to convince ourselves that this was a safe thing to do.

This also showed up a bug in our linker's handling of the `CAPALIGN` directive, which rounds a section's base and size up to the required alignment to be representable.
This was not working for sections that followed an explicit alignment directive.
Our stacks must be both at least 16-byte aligned *and* representable as capabilities.
This is now fixed.

So now we support stacks up to almost 64 KiB, a limitation imposed by the current loader metadata format rather than anything intrinsic to how the system operates after booting.
We could easily increase this limit but 64 KiB ought to be enough for anyone.

Performance on CHERIoT Ibex
---------------------------

The repository contains [a simple benchmark example](https://github.com/CHERIoT-Platform/cheriot-pqc/tree/main/examples/01.benchmark) that tries each of the operations and reports both the cycle time and stack usage.
The output on the CHERIoT Ibex verilator simulation is:

```
PQC benchmark: Starting: stack used: 224 bytes, cycles elapsed: 41
PQC benchmark: Generated ML-KEM key pair: stack used: 14304 bytes, cycles elapsed: 5143987
PQC benchmark: Encrypted secret pair with ML-KEM: stack used: 17440 bytes, cycles elapsed: 1773235
PQC benchmark: Decrypted secret pair with ML-KEM: stack used: 18464 bytes, cycles elapsed: 2176226
PQC benchmark: Compared results successfully for ML-KEM: stack used: 224 bytes, cycles elapsed: 414
PQC benchmark: Generated ML-DSA key pair: stack used: 46912 bytes, cycles elapsed: 3622132
PQC benchmark: Signed message with ML-DSA: stack used: 60544 bytes, cycles elapsed: 5391177
PQC benchmark: Verified message signature with ML-DSA: stack used: 44672 bytes, cycles elapsed: 3674071
PQC benchmark: Correctly failed to verify message signature with ML-DSA after tampering: stack used: 44672 bytes, cycles elapsed: 3673706
```

The ML-KEM encrypt (generate shared secret and encrypted version) and decrypt (recover shared secret from encrypted version) each use around 18 KiB of stack and run in around two million cycles.
CHERIoT Ibex should scale up to 200-300 MHz (though may be clocked lower for power reasons in some deployments), but even at 100 MHz that's 50 encryption or decryption operations per second.
Remember that this is an operation that typically happens when you establish a connection, then you use a stream cypher such as AES with the exchanged key.

The ML-DSA operations are slower and use a *lot* more stack space (almost 60 KiB for signing!).
But, even there, the performance is reasonable, under 4 M cycles.
This means that you can do 20 signature-verification operations per second at 100 MHz.

Even using ML-KEM for key exchange and ML-DSA for certificate validation in a TLS flow is unlikely to add more than a few tens of milliseconds to the handshake time, which is perfectly acceptable for the common use case for embedded devices.

In terms of code size, both are small.
The ML-KEM implementation is around 12 KiB, the ML-DSA implementation 18 KiB.
These both include a SHA3 (FIPS 202) implementation, so there's scope for code-size reduction on systems that need both, but 30 KiB of code isn't too bad.

Future plans
------------

The stack usage is very high.
Upstream has some plans to allow pluggable allocators, which will allow us to move a lot of this to the heap.
This is precisely the kind of use case that CHERIoT's memory-safe heap is great for: something needs 60 KiB of RAM for 4,000,000 cycles, but then doesn't need that RAM again for a long time.
That memory can then be used for something else, even in a mutually distrusting compartment.

Currently, the library builds are very thin wrappers around the upstream projects.
This is great as a building block, but we should make more use of CHERIoT features in the longer term.

Both ML-KEM and ML-DSA depend on SHA3 (FIPS 202).
Ideally, we'd factor that out as some common code, rather than carrying a copy in each library.
Similarly, the libraries provide an option to plug in your own SHA3 implementation.
This is likely to be a common hardware operation even for chips that don't have full PQC implementations, so we should expose this option in the build system.

Is it secure?
-------------

Security always depends on the threat model.

For signature validation, you don't have any secret data, just a public key, a document, and a signature.
The only concerns are whether there are weaknesses in the algorithm, or bugs, that would allow an attacker to substitute a different document for the same signature.
CHERIoT prevents memory-safety bugs, so this is concerned solely with logic errors.
The code upstream is checked against a set of test vectors that aim to trigger corner cases in the logic of the underlying implementation, so hopefully is secure in this way.

For signing or key exchange, you need to worry about the key leaking.
On a CHERI system, it's unlikely to leak explicitly, but may leak via side channels.
The [security section of the upstream projects](https://github.com/pq-code-package/mldsa-native?tab=readme-ov-file#security) discusses a number of techniques that they use to mitigate this kind of attack.

That's typically sufficient.
It's been recommended practice for embedded devices to have per-device secrets for a long time.
This means that leaking a key from one device doesn't compromise the device class, only that specific device.

For some very high-assurance use cases, that secret may matter and need to be robust against an adversary with physical access to the device.
Hardware encryption engines typically care about confidentiality breaches via power side channels and integrity breaches via glitch injection.
Power side channels are difficult to mitigate in software: the power requirements of multiplying two numbers together may depend on the number of carry bits set, for example.
They're much easier to mitigate in hardware, by simply doing the same calculation twice in parallel, once with the original inputs and once with the inputs permuted to have the opposite power characteristics.

Glitch injection takes the chip out of its specified power or frequency (or thermal) envelope and attempts to introduce bit flips, which can corrupt state in such a way that tamper with signing or leak a key.
These are also effectively impossible to mitigate in software because the software that's attempting the mitigation is vulnerable to the same glitches.
There are some compiler techniques that can make these harder, but they come with a high performance cost.

If power analysis and glitch injection are part of your threat model, the software implementations are not sufficient.
In this case you may also need to worry about someone removing the top of the chip and using a scanning-tunnelling electron microscope to read bits from non-volatile memory.
This used to require tens of thousands of dollars but is now much cheaper.
Devices that need to worry about this often have tiny explosive charges in the package to destroy the chip in cases of tampering.
If that's your threat model, hardware PQC implementations may not be sufficient, at least alone.

But if you care about attackers on the network being unable to compromise the security of the class of devices, even if they have a magical and imaginary quantum computer, then these should be sufficient.

