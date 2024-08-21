---
layout: post
title:  "£15,000 grants to prototype on CHERIoT"
date:   2024-08-21
categories: grant
author: David Chisnall
---

Digital Catapult today [announced a new Technology Access Programme (TAP) that covers CHERIoT](https://www.dsbd.tech/get-involved/technology-access-programme/).
The Digital Security by Design (DSbD) TAPs are intended to help companies prototype on CHERI systems, to build the CHERI ecosystem.
Prior TAPs have been restricted to Arm's [Morello](https://www.morello-project.org) prototype system.
This is the first that allows participants to build on CHERIoT.

The programme will provide lowRISC's excellent Sonata board to participants (these are also now [available to buy](https://www.mouser.co.uk/new/newae-technology/newae-sonata-one-dev-board/)).
This board makes it *incredibly* easy to get started with CHERIoT.
We've previously shown [that you can go from a standing start to running CHERIoT code in two minutes with Sonata](https://cheriot.org/fpga/ibex/2024/06/10/sonata-quick-start.html):

<video controls width="75%" style="margin-left: auto ; margin-right: auto; display: block">
  <source src="/images/Sonata Demo.mp4" type="video/mp4" />
  <p>Video showing how to start working in CHERIoT RTOS with Sonata.
     First clone the CHERIoT-RTOS repository from GitHub and open it in the dev container when prompted.
     Next, open a source file and observe that things like cross-references and inline API documentation work out of the box.
     Then run `xmake config --sdk=/cheriot-tools --board=sonata` in one of the projects to configure it.
     Finally, run `xmake` and `xmake run` to build and run.
  </p>
</video>

The basic environment gives you spatial and temporal memory safety out of the box, a privilege-separated RTOS, and a very easy mechanism for splitting your code into isolated compartments with fine-grained sharing.
You can try the [compartmentalisation exercise](https://github.com/CHERIoT-Platform/cheriot-rtos/blob/main/exercises/01.compartmentalisation/README.md) to see how easy it is to define compartment boundaries for fault isolation, protecting secrets, or mitigating compromises.
This exercise works in the simulator (you can even run it in a GitHub Code Space if you deploy one [from here](https://github.dev/cheriot-platform/cheriot-rtos)) and on Sonata.

The [CHERIoT prototype compartmentalised network stack](https://cheriot.org/rtos/networking/auditing/2024/03/08/cheriot-network-stack.html) runs on Sonata.
Between the compartmentalisation strategy employed and the foundational properties of the CHERIoT ISA, this provides a system where most bugs in the TCP/IP stack have little or no security impact.

Combined with Sonata's range of I/O facilities, this gives an excellent prototyping platform for secure IoT systems.
Anything that runs on Sonata should then be easy to port to [SCI Semiconductor's ICENI devices](https://www.scisemi.com/press-release-cheriot-ibex/) next year for commercial deployment at scale.

If you have a commercial IoT product that you want to be able to easily support in production for 10+ years, this TAP is a great way for you to explore how CHERIoT can help.

If you're considering participating in this TAP, and have any questions about the CHERIoT Platform, please don't hesitate to ask them in [GitHub Discussions](https://github.com/orgs/CHERIoT-Platform/discussions) or [our public Signal chat](https://signal.group/#CjQKIElxAs3t3MUEMOEmQEuMHRK4rErUk2xVeFzjAjFXAShzEhCK9qQwEMFKGLGZnCjrQ7zm).
