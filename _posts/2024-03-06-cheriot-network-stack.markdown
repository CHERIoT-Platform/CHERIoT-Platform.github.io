---
layout: post
title:  "Compartmentalising network stacks with CHERIoT"
date:   2024-03-08T12:00+00:00
categories: rtos networking auditing
author: "David Chisnall and Hugo Lefeuvre"
---

This week, we have made the ongoing work on a [CHERIoT network stack](https://github.com/CHERIoT-Platform/network-stack) public and we will be demonstrating it at the Digital Security by Design all-hands meeting next week.
Note that this is public to enable and encourage collaboration, it should absolutely *not* be considered production quality (yet).

The original public demos of CHERIoT had a network stack, with the FreeRTOS TCP/IP stack, mBedTLS, and a few other bits.
This was an early prototype implementation that was not designed to provide strong compartment boundaries, simply to demonstrate that it could work *at all*, mostly reusing existing code.
It involved several source-code changes in the FreeRTOS TCP/IP stack and, as an early prototype, was used to explore threat models for embedded network stacks.

Since then, a few things have changed in the upstream projects.
We've also learned a lot more about compartmentalisation and added a number of features to the RTOS from lessons learned the first port.
Our goals for the new network stack were somewhat more ambitious:

 - Zero code changes in the upstream projects.
   This meant improving the [FreeRTOS compatibility layer](https://cheriot.org/porting/freertos/2024/01/12/freertos-porting.html) so that we could run the FreeRTOS+TCP stack.
 - Providing a strong compartmentalisation model.
   Some of this has been refined during implementation.
 - Support TLS 1.2 and modern cypher suites.
   This was one of the main reasons for switching to BearSSL, because a configuration of mBedTLS with support for TLS 1.2 and elliptic curve cryptography instead of RSA was too large for our prototype platform.
 - Exposing simple APIs that work well for common use cases.
   The original version exposed the underlying APIs of the reused components at compartment boundaries but these APIs were not designed with untrusted callers in mind.
   We have started from a simple set of APIs tailored for typical IoT use cases and will expand them, rather than trying to secure the entire API surface of a mature library.

The project is open source and, if the APIs don't meet your requirements, you can always modify it (and, ideally, upstream support for new use cases to us!).
We made no code changes to the upstream projects and you can just put them all in a single compartment if that meets your security goals.
The upstream projects are:

 - FreeRTOS+TCP providing the TCP/IP stack.
 - FreeRTOS coreSNTP providing wall-clock-time services.
 - FreeRTOS coreMQTT providing MQTT.
 - BearSSL providing TLS 1.2.

There are several interesting bits of the CHERIoT platform that we've used in this stack that we'd like to highlight.

# Sealing gives opaque handles

On a traditional operating system, creating a socket gives you a handle or a file descriptor that is an index into some kernel-owned data structure.
In CHERIoT, each of the layers that provides a handle provides a sealed capability.
Sealing is a key part of the CHERI model.
Sealed capabilities are opaque values that can be unsealed by cannot be modified without first unsealing them.
You cannot use a sealed capability with a load, store, or jump instruction.
Any attempt to manipulate the capability itself will clear the tag.

To a C programmer, a sealed capability looks just like any other pointer.
You can embed it in structures, pass it around, and so on.
You can't use it to read or write the object that it refers to.
You can eventually hand it to the compartment that owns the matching capability with permit-unseal permission and they can turn it back into something that can be used like a pointer to a normal object.

This abstraction composes for multiple layers.
When you create a connected socket, you get a sealed capability that encapsulates the connection.
You can pass this around other compartments.
When you create a TLS connection, the TLS stack gives you a sealed capability to an object that encapsulates TLS state, including the sealed capability to the socket.

This combination means that TLS state is not visible to the caller and socket state is not visible to the TLS layer.
If you're using MQTT, then your handle to the MQTT connection state is a sealed capability to an object that contains a sealed pointer to the TLS state, making it sealed objects all the way down.
Each layer is isolated from the others and can see (or modify) only things explicitly passed as function arguments.

# Local capabilities and permissions enable safe temporary delegation

As we routinely discover when compartmentalising software: Isolation is easy, safe sharing is hard.
This is why CHERIoT was explicitly designed to facilitate a rich set of sharing.
No data from one layer in the stack is visible to other layers except when passed as a function argument but we often need to restrict things far more than simply sharing an entire object.

BearSSL, for example, reuses internal buffers.
TLS sits atop TCP and so is unaware of IP packets.
This means that the TLS stack must receive an entire message (which may be split over multiple packets) before it can complete the cryptographic integrity checks and pass it to the user.
This requires internal buffering.
The BearSSL APIs give a base and length for the part of the buffer that should either be sent over the network or into which new data should be received.
We precisely bound a capability to the length (reducing the length if necessary for the bounds encoding) and remove all permissions except load (when sending) or store (when receiving).

This means that, even if the TCP/IP stack is compromised (it's the thing that handles untrusted data from the network and so the most likely thing to be compromised), it cannot capture the pointer into the TLS stack's buffer because we have removed the global permission.
If we are receiving, it cannot read stale data from the buffer.
If we are sending, it can see only the cyphertext to send, not one byte more.

The no-capture guarantee also means that it cannot pass the buffer to another thread.
An attacker who compromises the thread that handles received packets, for example, must manipulate data in the TCP/IP stack to try to cause control-flow hijacking on the thread that sends or receives data for the TLS stack.
If that is successful, then the attacker still only has read access to data that would be flowing over the network or write access to a segment of the buffer into which the TLS stack expects to receive untrusted data.

# Mostly stateless compartments can isolate flows

The TLS compartment has 40 bytes of globals.
These are a mutex for static initialisation, a pointer to the state for the entropy source (larger on the FPGA than on real silicon because the FPGA does not have a hardware entropy source), and a pointer to the result of the last SNTP query that's used for getting time for checking certificate revocation lists.

All of the state associated with a TLS connection is encapsulated in that connection object.
This is unsealed during send and receive operations and is not stored anywhere other than the stack (which is zeroed on return).
This means that it is not reachable from any other thread, even one in the TLS compartment.
No amount of pointer chasing from one thread in the TLS compartment can lead to a different TLS connection context.

If an attacker manages to send a packet that contains a payload that compromises the TLS stack then this doesn't give them access to other flows.
The code in the compartment is immutable.
The may be able to influence entropy, which would harm the next rekeying event (this would not be possible on a non-prototype system with a realistic entropy source).
The may be able to alter the time and force you to accept expired certificates the next time you make a connection, but they don't get access to other existing connections.

# Compartmentalisation protects in multiple directions

Avoiding recurrences of Mirai-style botnets is one of my primary goals.
This means that I assume that the TCP/IP stack will be compromised and an attacker will try to both attack the application and mount distributed denial of service (DDoS) attacks on other systems.

The former is mitigated by the fact that the TCP/IP compartment is not allowed to make cross-compartment calls except to the firewall layer (to send Ethernet frames).
This means that it must wait for another compartment to call it and then provide some malicious data in response.
Most callers should not trust the data that they get from the network stack.
SNTP can validate signatures (it doesn't in the prototype but a production implementation should!) and TLS validates authentication codes on decrypted data, so anything provided by the network stack is already assumed to be malicious.
After all, it comes from the network, where the bad people live.

That doesn't mean that a compromised network stack is completely powerless.
Some of the calls take a malloc capability and the network stack could allocate memory to exhaust the quotas on these.
This is detectable though and a compartment that notices that the network stack is behaving badly may report this to a monitoring compartment that eventually reboots the device.

The situation in the opposite direction is more interesting.
Our prototype has an on-device firewall.
The firewall is a separate compartment so, unlike a conventional monolithic kernel, arbitrary-code execution in the network stack does not automatically imply the ability to modify the firewall.
The TCP/IP stack has the ability to close firewall holes (which it exercises as soon as it detects that a connection is closed to reduce its attack surface), but not open them.
The NetAPI compartment, which is responsible for creating new sockets for authorised callers, can open firewall holes and does so only when a connection is requested.
This means that a compromised TCP/IP stack cannot join DDoS attacks without further compromises.

This kind of defence in depth is what the CHERIoT platform was designed to provide.
Even the best TCP/IP stack is going to have occasional vulnerabilities, particularly over the 10+ year device lifetime that we're aiming for.
Our goal is to let device vendors handle these like any other bug by limiting the damage that an attacker can do.

In 2018 [the FreeRTOS+TCP stack was audited by Ori Karliner at Zimperium](https://www.zimperium.com/blog/freertos-tcpip-stack-vulnerabilities-details/) and 10 CVEs were reported.
Of these, 8 (slightly more than the 70% that we normally expect) were memory-safety bugs that are mitigated by CHERIoT simply as a result of recompiling.
On non-CHERI systems, several of these could lead to arbitrary-code execution, the remainder could lead at least to information disclosure.

One was a division-by-zero that caused a trap.
This can still happen, but putting it in a compartment means that the trap will, at worst, crash the network stack and not (for example) the real-time cyber-physical control system running on the same device.
The last was a DNS problem that is largely mitigated by our firewall strategy, which permits in- and outbound DNS packets only with the authorised DNS server and only while performing a name lookup.

# The shared heap enables zero-copy networking

Unlike most other platforms, CHERIoT provides a mechanism for object-granularity shared memory with a shared heap.
Each heap allocation is authorised by an allocation capability and counts against the associated quota.
Compartments may have zero, one, or many such capabilities and the system integrator can audit them.
The heap also allows callers to *claim* an allocation with another allocation capability.
This accounts it to both quotas and does not release the memory until both have freed it.

This mechanism makes it possible to `malloc` a packet buffer, have it flow through the network stack, and then to claim it in a receive call and transfer ownership of it to the caller.
The FreeRTOS+TCP stack supports this kind of zero-copy receive for UDP and we use it directly.
Receiving a UDP packet has a single copy from the network interface into the packet buffer and then flows it through the system.
Unfortunately the TCP/IP stack maintains ring buffers for TCP and so a zero-copy TCP receive gives a pointer into the ring buffer, which would not allow us to provide temporal safety.
Zero-copy TCP is not documented in FreeRTOS+TCP and so hopefully a future version will add official support and will maintain a list of packets that can be sent out for zero-copy receive.

Note that the claims mechanism can be used even with capabilities that are smaller than the object size.
This means that the network stack can hand out a pointer that's bounded to the payload of the UDP packet (without pointers in the buffer's header that refer to the descriptor, or anything in the header), and the caller can then free the packet at the end.

We hope to expand the zero-copy facilities in future iterations of the code.

# Capabilities and intentionality allow fine-grained auditing

The network stack uses a capability model.
To be able to create a socket that connects to a particular host, you must provide a capability that authorises the connection.
This is auditable using the [`cheriot-audit`](https://github.com/CHERIoT-Platform/cheriot-audit) utility.
For example, if you've just built the HTTPS example, then you can run the following command to see what compartments are allowed to talk to which hosts:

```sh
$ cheriot-audit --board ../../../cheriot-rtos/sdk/boards/ibex-arty-a7-100.json \
    --firmware-report build/cheriot/cheriot/release/03.https_example.json \
    --module ../../network_stack.rego \
    -q 'data.network_stack.all_connection_capabilities' | jq
[
  {
    "capability": {
      "connection_type": "UDP",
      "host": "pool.ntp.org",
      "port": 123
    },
    "owner": "SNTP"
  },
  {
    "capability": {
      "connection_type": "TCP",
      "host": "example.com",
      "port": 443
    },
    "owner": "https_example"
  }
]
```

This shows that the `SNTP` compartment can talk to pool.ntp.org on port 123 with UDP and the `https_example` compartment can talk TCP to example.com on port 443.
This kind of query can be embedded in a policy that you use with `cheriot-audit` to drive code-signing decisions.

The [`network_stack.rego`](https://github.com/CHERIoT-Platform/network-stack/blob/main/network_stack.rego) file contains a basic policy for the network stack itself.
For the HTTPS example, you can query that policy by simply passing it the name of the Ethernet device from the board-description file:

```
$ cheriot-audit --board ../../../cheriot-rtos/sdk/boards/ibex-arty-a7-100.json \
    --firmware-report build/cheriot/cheriot/release/03.https_example.json \
    --module ../../network_stack.rego \
    -q 'data.network_stack.valid("kunyan_ethernet")' 
true
```

If this outputs `true`, then the firmware image enforces the policy.
Again, this is in early development and so this doesn't guarantee that the stack is secure, but it does guarantee that:

 - The network device is accessible only by the firewall.
 - The private APIs for the different compartments in the network stack are accessible only from the compartments that are supposed to call them.
 - The threads for the firewall (incoming packets) and TCP/IP (state machines and outgoing packets) exist and have sensible stack sizes.
 - All sealed capabilities that claim to be connection capabilities have valid contents.

This auditing works even in the presence of binary-only compartment files and so supports firmware images that incorporate components from multiple vendors.

