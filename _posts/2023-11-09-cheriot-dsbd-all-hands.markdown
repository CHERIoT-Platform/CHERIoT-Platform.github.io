---
layout: post
title:  "CHERIoT at the Digital Security by Design All Hands meeting"
date:   2023-11-09T13:45+00:00
categories: conference
author: "David Chisnall"
---

![The DSbD event was well attended](/images/2023-11-08-dsbd-all-hands-talks.jpg)

CHERIoT was well represented yesterday at the [Digital Security by Design (DSbD) All-Hands meeting](https://www.dsbd.tech/event/dsbd-all-hands-2/) in Manchester.
This was a gathering of folks funded by, or otherwise contributing to, the DSbD programme.

![SCI Semiconductor presented the MICRO poster](/images/2023-11-08-dsbd-all-hands-sci.jpg)

[SCI Semiconductor](https://www.scisemi.com), which aims to ship CHERIoT silicon and a supported software stack to customers next year, was presenting the poster from our MICRO paper as well as a demonstration of the CHERIoT RTOS running on CHERIoT Ibex on an Arty A7 100T FPGA.

The A7 is, for now, the easiest hardware for individuals wanting to try out the CHERIoT platform.
More details on this to follow in a later post.
Currently, there's no way of updating the software other than reprogramming the FPGA, so it's not a recommended flow, but this should be improved very soon.

![SCI Semiconductor demonstrated the CHERIoT platform in FPGA](/images/2023-11-08-dsbd-all-hands-fpga.jpg)

lowRISC also presented a poster on their [Sunburst Project](https://www.sunburst-project.org), funded by the DSbD programme, which aims to ship a custom board with an FPGA that can run the CHERIoT Ibex, allowing easy development on the CHERIoT platform.

![lowRISC showed a poster about the Sunburst FPGA](/images/2023-11-08-dsbd-all-hands-lowrisc.jpg)

Finally, in the afternoon we presented a compartmentalisation workshop ([slides](/papers/2023-11-08-Compartmentalisation-Workshop.pptx), [pdf slides](/papers/2023-11-08-Compartmentalisation-Workshop.pdf)).
This included an overview of compartmentalisation and a demo of Dapeng Gao's excellent work on retrofitting compartment boundaries around shared libraries.

It finished with an exercise for the attendees to try adding compartmentalisation to a simple example on CHERIoT:

![A room full of people tried to compartmentalise software with CHERIoT](/images/2023-11-08-dsbd-all-hands-workshop.jpg)

In this exercise, a [JavaScript interpreter](https://microvium.com) on the device simulates an attacker who has been able to gain arbitrary code execution via a code reuse attack.
The firmware reads JavaScript bytecode over the UART and executes it.
We've exposed some functions via FFI that allow arbitrary memory reads and capability manipulation.

In the initial version, there's no compartmentalisation and so an attacker can leak a secret or crash the firmware.
Over three exercises, you have to:

1. Ensure that the secret is protected (enforcing confidentiality).
2. Prevent crashes in the JavaScript from corrupting any other state (enforcing integrity).
3. Prevent crashes in the JavaScript from leaking, and then exhausting, memory (protecting availability).

[The exercise is in the CHERIoT RTOS repository](https://github.com/microsoft/cheriot-rtos/tree/main/exercises/01.compartmentalisation) if you missed the workshop and want to try it.
For the workshop, we recommended that attendees use [GitHub Codespaces](https://github.com/features/codespaces), which allow you to work in a web browser talking to a container managed by GitHub.
Creating a codespace for the CHERIoT RTOS repository will use our dev container automatically, which includes the toolchain and the Ibex (cycle-accurate) and Sail (instruction) simulators.

