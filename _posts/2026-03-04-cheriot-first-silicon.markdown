---
layout: post
title:  "First CHERIoT Silicon!"
date:   2026-03-04
categories: silicon
author: David Chisnall
---

<img alt="ICENI on a development board" width="80%" style="margin-left:auto;margin-right:auto;display:block" src="/images/2026-03-04 iceni.png">

Most CHERIoT work to date has been done on software or FPGA simulations.
We have several such implementations: The executable model built from our [formal ISA specification](https://github.com/CHERIoT-Platform/cheriot-sail), the [MPact simulator from Google](https://mpact.googlesource.com/mpact-cheriot/), [Microsoft's CHERIoT SAFE FPGA target for the Arty A7](https://github.com/microsoft/cheriot-safe), and of course lowRISC's beautiful [Sonata FPGA board, which is designed to simulate CHERIoT systems](https://www.mouser.co.uk/new/newae-technology/newae-sonata-one-dev-board).
These were always intended to be developing and prototyping systems, so I'm delighted to announce that SCI Semiconductor has the first silicon CHERIoT implementation.

[ Conflict disclaimer: I am a co-founder of SCI Semiconductor. ]

The dev board pictured above contains one of the first batch of ICENI chips to come back from the fab.
This is a complete CHERIoT system, with all of the core CHERI properties (spatial memory safety, no pointer injection, and so on) along with all of the CHERIoT extensions that provide deterministic use-after-free protection, auditable control over interrupt state, and everything that we need for an aggressively compartmentalised RTOS.

This chip uses the CHERIoT Ibex core, running at up to 250 MHz, and includes a few feature that accelerate temporal safety, improve interrupt determinism, and so on.
These build on top of all of the benefits of any CHERIoT implementation: deterministic mitigation of memory safety bugs from simple buffer overflows up to use-after-free, fine-grained compartmentalisation, and a programming model co-designed with both the ISA and the software stack to provide a tiny TCB.
Anything that works on CHERIoT SAFE or Sonata should be very easy to port to ICENI for production use.
Anything that runs on the software simulators should just work.

We'll be showing the chips at [Embedded World (Stand 4A - 131)](https://www.embedded-world.de/en) next week and at [CHERI Blossoms](https://cheri-alliance.org/events/cheri-blossoms-conference-2026/) a couple of weeks later.
From tomorrow, one will also be on display in the CHERI 15th anniversary exhibit in the Cambridge Computer Laboratory.

Aside: The [Iceni tribe](https://www.mouser.co.uk/new/newae-technology/newae-sonata-one-dev-board) were one of the pre-Roman tribes in Britain and are famous for their chariots (though more due to [this statue](https://en.wikipedia.org/wiki/Boadicea_and_Her_Daughters) than historical fact).
I am only partially to blame for the bad puns in the naming.
