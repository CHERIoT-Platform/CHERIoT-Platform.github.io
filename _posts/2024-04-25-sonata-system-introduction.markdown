---
layout: post
title:  "Sonata at SOOCon24"
date:   2024-04-17T16:00+00:00
categories: platform conference
author: "Marno van der Maas"
---

Dr Marno van der Maas gave [a talk at the State of Open Conference 2024 (SOOCon24)](https://stateofopencon2024.sched.com/event/1Xl43) on Sonata on 6 February 2024 at 1:30pm in London. The title of his talk is *Sonata: low-cost CHERI hardware for embedded systems*. Here's the [handout for the presentation](https://www.sunburst-project.org/pdf/SOOCon24_Sonata_handout.pdf) and a [video recording](https://redirect.invidious.io/watch?v=_FGfRwKSfDg).

Sonata is a low cost development board for investigating CHERI security enhancements. This talk introduces Sonata as the platform to drive CHERIoT adoption forward in the embedded space. CHERI research benefited from open source projects such as FreeBSD and LLVM. CHERIoT Ibex exists because of open architectures (the [RISC-V ISA](https://riscv.org/technical/specifications/), [version 9 of the CHERI ISA](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-987.pdf) and the [CHERIoT ISA](https://www.microsoft.com/en-us/research/publication/cheriot-rethinking-security-for-low-cost-embedded-systems/)) and the open silicon [Ibex core](https://github.com/lowRISC/ibex). The fully open nature of Sonata (design, verification, board layout, software, etc.) will in turn provide many opportunities for others to continue building on this work. Open source can work well to drive innovation and Sonata is playing its part in the CHERI story here.

The Sonata boards allows many different types of users to investigate CHERI features and to support this there are many common expansion ports and features that an embedded user would usually expect. Sonata is all about being usable, debuggable, connectable, extendable and interactive. To achieve all of this, we need a balance of IP blocks. There is also a big focus on configurability. For example the pin multiplexer can be used to drive an LED by a pulse width modulator instead of a GPIO. And the number of I2C, SPI and UART devices is configurable. Further use-cases can be achieved using the extension headers provided. You can buy off-the-shelf boards to add wireless functionality for example. The Raspberry Pi Hat, Arduino Shield and mikroBus Click have connectors on the top of the board. QWIIC uses I2C to daisy chain boards together, which has two connectors on the side. There are two PMOD connectors on the side as well as RS-232 and RS-485 connectors. Finally, there are interactive features like an LCD display and a 5-way joystick.

Please read more details in the [Sonata documentation](https://lowrisc.org/sonata-system/) and the [Sonata repository](https://github.com/lowRISC/sonata-system). Sonata is part of the [Sunburst project](https://www.sunburst-project.org/) and supported by [UKRI](https://www.ukri.org/) and [DSbD](https://www.dsbd.tech/) under Grant Number 107540.
