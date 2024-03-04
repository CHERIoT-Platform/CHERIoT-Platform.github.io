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

 - The  [formal model of the CHERIoT ISA](https://github.com/microsoft/cheriot-sail).
   This is used to build an executable simulator and to prove properties of both the ISA and of implementations.
 - The [CHERIoT RTOS](https://github.com/microsoft/cheriot-rtos), a clean-slate RTOS designed to take advantage of CHERIoT features.
   This provides the compartment model, a heap that can be safely shared across mutually distrusting compartments, and a host of other features.
 - [CHERIoT LLVM](https://github.com/CHERIoT-Platform/llvm-project) provides the toolchain for building the RTOS and other software that runs atop it.
 - [CHERIoT-Audit](https://github.com/CHERIoT-Platform/cheriot-audit) provides tooling for auditing the isolation properties of CHERIoT firmware images.
 - The [CHERIoT Ibex](https://github.com/microsoft/cheriot-ibex), an area-optimised core that implements the ISA.
   This is very slightly larger than the Ibex with a 16-element Physical Memory Protection unit, yet provides object-granularity memory safety and scales to a number of compartments bounded only by available memory.
 - The [CHERIoT small and fast FPGA emulator](https://github.com/microsoft/cheriot-safe) platform.
   This provides a set of peripherals such as a UART and interrupt controllers that provide a minimal useful integration of the Ibex.

The [CHERIoT dev container](https://github.com/orgs/CHERIoT-Platform/packages/container/package/devcontainer) includes the toolchain, the simulator built from the formal model, and a verilator simulation of the Ibex.
This can be used explicitly via Docker / Podman or by opening the RTOS repository in Visual Studio Code or another editor that supports dev containers.

