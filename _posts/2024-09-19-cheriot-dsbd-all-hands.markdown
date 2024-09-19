---
layout: post
title:  "CHERIoT at the Digital Security by Design All Hands meeting"
date:   2024-09-19
categories: conference
author: "David Chisnall"
---

Several companies presented CHERIoT-related things at the [Digital Security by Design](https://www.dsbd.tech) all-hands meeting yesterday!

[lowRISC](https://lowrisc.org), whose [Sonata](https://cheriot.org/fpga/ibex/2024/06/10/sonata-quick-start.html) board was used by all of the demos, presented a demonstration of an automotive system where a bug in the volume control would overwrite the speed controller value (on a non-CHERI system).
The source for this [is in the Sonata software repo](https://github.com/lowRISC/sonata-software/tree/main/examples/automotive), as is the [snake example](https://github.com/lowRISC/sonata-software/tree/main/examples/snake) that lowRISC also showed.

<img alt="lowRISC presented an automotive demonstrator" width="50%" style="margin-left:auto;margin-right:auto;display:block" src="/images/2024-09-19-lowRISC-demo.jpeg">

[ConfiguredThings](https://www.configuredthings.com) presented an extended version of the [configuration management demonstration](https://cheriot.org/security/philosophy/2024/07/30/configuration-management.html) that they've previously contributed to the project.
The updated version integrated the CHERIoT network stack to talk to their back-end secure configuration management system.
The code for [the original version of their demo](https://github.com/CHERIoT-Platform/cheriot-demos/tree/main/configuration_broker) is open and the network-connected version should appear in the same place soon.

<img alt="ConfiguredThings presented CHERIoT talking to their back-end system" width="50%" style="margin-left:auto;margin-right:auto;display:block" src="/images/2024-09-19-configuredthings-demo.jpeg">

This showed how a CHERIoT system can provide additional defence in depth.
Each configuration block from the server was parsed in a separate compartment, so bugs in the JSON parsing are not exploitable.
The worst that can happen is that an invalid configuration update is ignored.
CrowdStrike provided a good demonstration of how bad this can be without CHERI.

Finally, we at [SCI Semiconductor](https://www.scisemi.com) presented a demonstration of the network-stack restart work that we released over the summer.
This ran on Sonata, but (as with the other demonstrators) will be trivial to port to our [ICENI CHERIoT chips, which are expected early next year](https://www.scisemi.com/press-release-cheriot-ibex/).
This showed a simple multi-colour light that was connected to the Internet via MQTT.
The CHERIoT network stack runs the FreeRTOS TCP/IP stack ('[FreeRTOS+TCP](https://github.com/FreeRTOS/FreeRTOS-Plus-TCP)') in a compartment.
We introduced a memory-safety bug into this code, which forms a key part of the attack surface (it's the thing that has to process packets that come from the Internet, where all of the bad people live).
When this is triggered, we see a CHERI exception on Sonata's CHERI fault LEDs and the network connection is dropped.
The TCP/IP compartment is then restarted automatically and the application code resumes:

<video controls width="75%" style="margin-left: auto ; margin-right: auto;      display: block">
  <source src="/images/Hugh the Lightbulb.mp4" type="video/mp4" />
  <p>Video showing Hugh the Lightbulb, an Internet-connected multicolour light.
    The video shows an Android app controlling the CHERIoT code and demonstrates that a memory-safety bug in the TCP/IP stack does not crash the system, but is caught and the TCP/IP stack gracefully recovers.
  </p>
</video>

The [code for this demo](https://github.com/CHERIoT-Platform/cheriot-demos/tree/main/HughTheLightbulb) is available.
Note that there's *nothing* in the application-specific part of the code related to the TCP/IP stack crashing.
From the perspective of a consumer of the TCP/IP APIs, sockets just return a disconnection error.
The normal reconnection paths then succeed once the TCP/IP stack has been restarted.

