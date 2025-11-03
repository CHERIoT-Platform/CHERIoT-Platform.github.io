---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: post 
---

<img src="images/fpga.jpeg" alt="FPGA running CHERIoT Ibex">

## Latest news

{% assign posts = site.posts %}
{% for item in posts limit:4 %}
  {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
 - [*{{ item.title }}* - {{ item.date | date: date_format }}]({{ item.url }})
{% endfor %}

[More...](news)

## About

The Capability Hardware Extension to RISC-V for IoT (CHERIoT) platform was originally developed at Microsoft and is now part of an effort spanning multiple companies.
It builds on top of [CHERI](https://cheri-cpu.org) to provide a solid foundation for secure embedded devices.
CHERI provides referential integrity (pointers cannot be forged), spatial memory safety (pointers carry bounds that cannot be extended), call gates, and so on.

CHERIoT extends this with a complete platform providing deterministic use-after-free protection, a lightweight compartment model, lexically-scoped delegation of objects across compartment calls, and many more benefits.

The CHERIoT project comprises several repositories:

 - The  [platform specification, including the formal model of the CHERIoT ISA](https://github.com/CHERIoT-Platform/cheriot-sail).
   This is used to build an executable simulator and to prove properties of both the ISA and of implementations.
   [The under-development (draft) version of the specification](https://cheriot.org/cheriot-sail/cheriot-architecture.pdf) is built in CI.
    The [1.0 (current) release of the specification](https://github.com/CHERIoT-Platform/cheriot-sail/releases/download/v1.0/cheriot-architecture-v1.0.pdf) is also available.
 - The [CHERIoT RTOS](https://github.com/CHERIoT-Platform/cheriot-rtos), a clean-slate RTOS designed to take advantage of CHERIoT features.
   This provides the compartment model, a heap that can be safely shared across mutually distrusting compartments, and a host of other features.
 - [CHERIoT LLVM](https://github.com/CHERIoT-Platform/llvm-project) provides the toolchain for building the RTOS and other software that runs atop it.
 - [CHERIoT-Audit](https://github.com/CHERIoT-Platform/cheriot-audit) provides tooling for auditing the isolation properties of CHERIoT firmware images.
 - The [CHERIoT Ibex](https://github.com/microsoft/cheriot-ibex), an area-optimised core that implements the ISA.
   This is very slightly larger than the Ibex with a 16-element Physical Memory Protection unit, yet provides object-granularity memory safety and scales to a number of compartments bounded only by available memory.
 - The [CHERIoT small and fast FPGA emulator](https://github.com/microsoft/cheriot-safe) platform.
   This provides a set of peripherals such as a UART and interrupt controllers that provide a minimal useful integration of the Ibex.

The [CHERIoT dev container](https://github.com/orgs/CHERIoT-Platform/packages/container/package/devcontainer) includes the toolchain, the simulator built from the formal model, and a verilator simulation of the Ibex.
This can be used explicitly via Docker / Podman or by opening the RTOS repository in Visual Studio Code or another editor that supports dev containers.

## Getting started

It's very easy to start developing for CHERIoT with either an [Arty A7](https://digilent.com/reference/programmable-logic/arty-a7/start) or [Sonata](https://www.sunburst-project.org) FPGA board, or with no hardware and using a simulator.
The Sonata boards (which, unlike the A7, are designed specifically for prototyping CHERIoT-based systems) are [now available to buy from Mouser](https://www.mouser.co.uk/ProductDetail/NewAE/NAE-SONATA-ONE?qs=wT7LY0lnAe1k3dLvmL42Eg%3D%3D).

The [CHERIoT Getting Started Guide](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/main/docs/GettingStarted.md) explains how to build and run code for all of these.

We also have a [From zero to CHERIoT in two minutes with Sonata](https://cheriot.org/fpga/ibex/2024/06/10/sonata-quick-start.html) blog post hat explains how to start building and running code on Sonata in about the same amount of time as it will take you to get the board out of its box and plug it in.


## Where to ask questions

We use [GitHub Discussions](https://github.com/orgs/CHERIoT-Platform/discussions) for general queries about CHERIoT.
This is persistent and searchable (without an account) and so a good place to ask questions that someone else may want to know the answer to.

We also have a [public Signal chat](https://signal.group/#CjQKIElxAs3t3MUEMOEmQEuMHRK4rErUk2xVeFzjAjFXAShzEhCK9qQwEMFKGLGZnCjrQ7zm).
The Signal chat is intended for live discussions and automatically deletes messages.
We encourage participants to write up the results of any discussions there in documentation, GitHub Discussions, Issues, or somewhere else that's searchable.
You can join the group from your phone by scanning this QR code:

<p style="text-align: center; margin-left: auto; margin-right: auto">
<img src="images/signal-group-qr-code.png" width="100pt" alt="QR Code for joining the CHERIoT Signal public chat">
</p>
