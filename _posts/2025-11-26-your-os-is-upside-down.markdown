---
layout: post
title:  "Your RTOS is upside down"
date:   2025-11-26
categories: rtos philosophy
author: "David Chisnall"
---


In the last month or so (partly as a result of going to SOSP), I've seen a lot of architecture diagrams for operating systems and one thing has struck me about all of them: they put the device drivers in the wrong place.
Here, for example, is TockOS:

![TockOS architecture diagram, showing the bottom layer containing all of the device drivers](/images/tock-architecture.png)

TockOS is implemented in Rust and has safety as a priority.
I generally regard it as the gold standard for an RTOS that is forced to operate under the constraints of working with existing hardware.
But note where the drivers are: In the kernel, right at the bottom.

Here's a similar diagram from the Zephyr RTOS:

![Zephyr architecture diagram, showing the bottom layers containing all of the device drivers](/images/zephyr-architecture.png)

Here, there is some separation between drivers and the core of the kernel, but the default configuration runs with no privilege separation.
Indeed, the [Zephyr Security Overview](https://docs.zephyrproject.org/latest/security/security-overview.html) says:

> The security architecture is based on a monolithic design where the Zephyr kernel and all applications are compiled into a single static binary.
> System calls are implemented as function calls without requiring context switches. 

In fact, the only exception I've seen to this recently is LionsOS, a multiserver system built on top of the formally verified seL4 microkernel:

![LionsOS architecture diagram, showing the device drivers in userspace processes](/images/lionsos-architecture.svg)

This runs all of the device drivers in unprivileged contexts.
Unfortunately, seL4 assumes an MMU and so is not feasible for small embedded devices (seL4 uses more memory to hold page tables than a lot of CHERIoT firmware images use in total).

Drivers are attack surface
--------------------------

Device drivers, at least for I/O devices, are code that interacts with the outside world.
An attacker trying to compromise a device has a much easier job if they don't need to take the chip apart.
The easiest attacks to mount are ones that work from an across the network.
The next easiest are ones that drive local I/O, which may be reachable remotely via other paths.

Drivers for I/O devices, by definition, make decisions based on the values that they read from the device.
These operations include mapping error codes to some type-safe enumeration (which can fail if the hardware is buggy), or using the values to index into other structures.
These are hard to get right, even in a memory-safe language, because they often sit below the language's abstraction layer.
Microsoft estimates that [70% of Windows crashes are caused by bugs in device drivers](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/stop-code-error-troubleshooting#what-causes-stop-errors).

Any bug in a driver for an I/O device is a useful building block for an attacker.
This problem is made *much* worse if the device driver is in a privileged component.
For example, at the end of last month there were [three](https://app.opencve.io/cve/CVE-2025-10456) [bluetooth](https://app.opencve.io/cve/CVE-2025-10458) [CVEs](https://app.opencve.io/cve/CVE-2025-7403) in Zephyr that all could lead to compromise and, if the Bluetooth stack is not privilege separated, can lead to arbitrary code execution by an attacker who gets within a few meters of the device (or compromises another Bluetooth-enabled device nearby).

Device drivers do abstraction and multiplexing
----------------------------------------------

This design results from conflating the two functions of a device driver.

A device driver has to provide an abstraction over a particular device.
Sometimes this happens in multiple layers.
For example, an Ethernet device may have an abstraction for sending and receiving Ethernet frames, but this is then the foundation for a further abstraction layer for sending IP packets, which is then used to expose TCP streams and UDP datagrams.

A device driver often *also* has to provide some secure multiplexing.
For example, two mutually distrusting components may be allowed to create sockets for different TCP connections that flow over the same Ethernet device.
Or they may be allowed to talk to two different USB bus endpoints via the same USB controller.

The first of these requirements is a *software engineering* problem.
The second is primarily a *security* problem, but typically the multiplexed abstractions need to be device independent and so it's *also* a software-engineering problem.

In embedded development, there's often a distinction between a hardware-abstraction layer (HAL) and a driver, with the former providing only the abstraction and the latter also providing multiplexing.
This is a useful distinction because, in a lot of cases, embedded systems have a *single consumer* for a device.
For example, you may have multiple SPI or I<sup>2</sup>C interfaces on a device, but each one is used for a single purpose.
It is convenient to be able to write software to talk to a SPI device without having to know exactly *which* SPI controller this chip uses, but you don't need to handle safely sharing those pins with other components.

CHERIoT RTOS distrusts drivers
------------------------------

In CHERIoT RTOS, the core platform provides device abstractions that meet the earlier definition of a HAL: they provide abstractions over classes of device, but do not attempt to provide security.
The RTOS also provides a trivial way of auditing which compartments can access which devices, so that you can ensure that devices are not accessible to compartments that are not trusted to interface with them.

The platform's device code runs within whatever compartment you instantiate it in.
It has no elevated privileges *except* the MMIO region(s) that you explicitly pass it for talking to a particular device.

This makes it easy to support both bespoke and reusable security models.
If you need to share a device between two compartments with some secure multiplexing based on a custom policy, you can do that by instantiating the driver in a compartment and exposing APIs to the two others.

Sometimes, the desired abstractions are reusable.
For example, the CHERIoT network stack is assembled out of the following compartments:

<pre class="mermaid">
graph TD
  Network
  subgraph Firewall["On-device firewall"]
    DeviceDriver["Device Driver"]
  end
  TCPIP["TCP/IP"]:::ThirdParty
  User["User Code "]
  NetAPI["Network API"]
  DNS["DNS Resolver"]
  SNTP:::ThirdParty
  TLS:::ThirdParty
  MQTT:::ThirdParty
  DeviceDriver <-- "Network traffic" --> Network
  TCPIP <-- "Send and receive Ethernet frames" --> Firewall
  DNS <-- "Send and receive Ethernet frames" --> Firewall
  NetAPI -- "Perform DNS lookups" --> DNS
  NetAPI -- "Add and remove rules" --> Firewall
  TLS -- "Request network connections" --> NetAPI
  TLS -- "Send and receive" --> TCPIP
  NetAPI -- "Create connections and perform DNS requests" --> TCPIP
  MQTT -- "Create TLS connections and exchange data" --> TLS
  User -- "Create connections to MQTT server and publish / subscribe" --> MQTT
  MQTT -- "Callbacks for acknowledgements and subscription notifications" --> User
  SNTP -- "Create UDP socket, authorise endpoints" --> NetAPI
  SNTP -- "Send and receive SNTP (UDP) packets" --> TCPIP
  TLS -- "Request wall-clock time for certificate checks" --> SNTP
  style User fill: #5b5
  classDef ThirdParty fill: #e44
</pre>

Note that the driver for the Ethernet device is instantiated in the firewall compartment.
What happens if an attacker gets arbitrary-code execution here?
They could mount a denial of service attack (refuse to forward Ethernet frames in or out).
They could tamper with Ethernet frames.

This sounds bad but the rest of a network stack already has to assume that things like this can happen.
Packets coming over the network are intrinsically untrusted.
The TCP/IP stack has to assume that they may be malicious.
It isn't always good at this.
The FreeRTOS TCP/IP stack that we use has had 15 CVEs disclosed since it was released, but our compartmentalisation strategy mitigates all of them.
By placing the parts of the system that are exposed to an attacker in the *least*, not most, trusted places, we make it easy to build secure systems.

<script src="https://cdn.jsdelivr.net/npm/mermaid@10.9.1/dist/mermaid.min.js"></script>
