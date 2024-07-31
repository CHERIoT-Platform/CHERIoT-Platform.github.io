---
layout: post
title:  "CHERIoT projects moving into the CHERIoT Platform org"
# Fix this and the file name before publishing!
date:   2024-07-31
categories: rtos sail
author: David Chisnall
---

I am pleased to announce that, today, Microsoft has transferred the [CHERIoT Sail](https://github.com/CHERIoT-Platform/cheriot-sail) and [CHERIoT RTOS](https://github.com/CHERIoT-Platform/cheriot-rtos) repositories into the CHERIoT-Platform GitHub organisation.
GitHub *should* redirect everything from the old locations, but it's probably a good idea to update URLs in bookmarks, git clones, and so on.

CHERIoT began as a research project in Microsoft Research Cambridge, as part of Microsoft's work on the [Digital Security by Design Programme](https://www.dsbd.tech).
The project's goal was to explore several aspects of CHERI system design, specifically:

 - What can you do with software abstractions if you can assume CHERI?
 - How can you scale down the work on temporal safety to tiny embedded devices?
 - How can you remove the need for bolted-on security extensions by creating a holistic hardware-software security model around an assumption of memory safety?

As a research project, it was a resounding success.
[We demonstrated](https://www.microsoft.com/en-us/research/publication/cheriot-complete-memory-safety-for-embedded-devices/) that a foundation of CHERI let you build tiny cores that provided a step change in the baseline security guarantees, along with a simple programmer model, in exchange for a tiny power and area overhead.

Microsoft; however, is not a microcontroller vendor and, for CHERIoT to be useful, it needs broad ecosystem support.
This ecosystem is forming, with several companies making significant early contributions.
Google has contributed, and we have merged, improvements to the ISA specification and various parts of the software stack.
SCI Semiconductor is working to ship commercial CHERIoT SoCs to customers next year.
lowRISC has built the [Sonata FPGA platform](https://www.sunburst-project.org) for prototyping CHERIoT devices.
Folks at [Configured Things](https://www.configuredthings.com) have written a [fantastic demo application](https://github.com/CHERIoT-Platform/cheriot-demos/tree/main/configuration_broker) (see [yesterday's blog for more information](/philosophy/2024/07/30/configuration-management.html)).

At this point, CHERIoT is no longer a research project, it is an open source foundation supported by multiple vendors and these repositories' moves reflect this fact.
The CHERIoT-Platform organisation is now a centralised landing pad for anything CHERIoT related.

The two repositories that have moved today are the ISA specification and the RTOS.
These are core parts of the platform.

The [CHERIoT Sail repository](https://github.com/CHERIoT-Platform/cheriot-sail) contains the ISA specification, including both the formal model and the [prose descriptions](https://cheriot-platform.github.io/cheriot-sail/cheriot-architecture.pdf).
This executable formal model can be used to prove properties of the ISA, verify that implementations conform to the specification, verify properties of software running on CHERIoT cores, and also build our golden model simulator.
This repository contains everything that you need to be able to build a CHERIoT core and validate that it really implements the ISA.
I believe that formal verification is a key part of any secure system.
Formal specification is the foundation on which formal verification is built and I'm excited by the results we've seen so far from groups building on this.

The CHERIoT RTOS repository contains the core parts of the software stack.
CHERIoT is unusual in being a complete hardware-software stack, where the hardware, programmer model, and software were all designed together.
The CHERIoT RTOS is the embodiment of the compartmentalisation model that the CHERIoT ISA was designed to support (and, conversely, the CHERIoT ISA was designed to run the CHERIoT RTOS).
Although you can run other operating systems on CHERIoT, you will get the most benefit from using the software stack that was designed around the guarantees that the hardware provides.

Readers who have been following the project for a while may notice one omission.
Microsoft is *not* transferring the [CHERIoT Ibex](https://github.com/Microsoft/cheriot-ibex) core.
This remains an open-source core that is rapidly approaching commercial quality and is the first core that can be used to build a CHERIoT Platform.
We want to make it clear that it is not *the* CHERIoT core, merely the *first* commercial-quality CHERIoT core.

When we originally prototyped CHERIoT, we built two hardware implementations.
In addition to the Ibex, which aimed at production use, we also prototyped the CHERIoT ISA on the [CHERI Flute](https://github.com/CTSRD-CHERI/Flute) processor.
Ibex was optimised more for power and area, Flute more for performance.
Ibex was a 3-stage pipeline with a 33-bit memory bus (requiring two cycles to load a capability).
Flute was a 5-stage pipeline with a 64-bit memory bus (loading capabilities in a single cycle).

We did this to ensure that the CHERIoT ISA was not over-fitted for one particular microarchitecture.
We expect it to scale from (tiny!) things the size of Ibex up to quite large microcontrollers.

The Ibex, which vital to the ecosystem in demonstrating the viability of CHERIoT, was never intended to be the *only* CHERIoT core.
We welcome additional implementations (open and proprietary) and have no wish to bless any particular implementation and exclude others.

All of that said, the community engagement with CHERIoT Ibex has been amazing.
It has been the focus of several formal verification efforts and I am very happy to see contributors from across industry and academia working to improve it.
We will continue to include a cycle-accurate simulator of the Ibex in the [CHERIoT Dev Container](https://github.com/orgs/CHERIoT-Platform/packages/container/package/devcontainer) and will add any other implementations that are available under licenses that permit redistribution.

The CHERIoT Platform organisation currently has two maintainers: myself and Yucong Tao (Microsoft), and may add more in the future.
Burdening a young project with too much bureaucracy early on is a sure way to kill it and so the administration is currently very light.
This forms a nucleus around which we can form a CHERIoT Foundation if such a thing is desirable in the future.

As part of the repository transfer, I have signed a legal agreement with Microsoft that the components in these repositories will remain open source and under permissive licenses.
This agreement will also bind future maintainers of the CHERIoT Platform.
We are committed to providing an open ecosystem for CHERIoT devices.
Device vendors are legally permitted to take as much or as little of this as they wish, and modify it as much as they wish.
We encourage anyone with downstream changes that might benefit others to consider contributing them upstream.

Anyone wanting long-term commercial support for the CHERIoT software stack should contact [SCI Semiconductor](https://www.scisemi.com).
