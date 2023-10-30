---
layout: post
title:  "CHERIoT at MICRO 2023"
date:   2023-10-30T09:03+00:00
categories: ibex flute architecture publication
---

This week, some of the CHERIoT team will be at [MICRO 2023](https://microarch.org/micro56/index.php) presenting the first paper about the CHERIoT platform:
{% cite cheriotmicro2023 %}.
This paper describes the CHERIoT ISA extension and the microarchitectural techniques used to make it fast, with low area overhead.

The paper describes how the same architecture can be supported on cores in two interesting places in the microcontroller design space.
Our initial prototype implementation, described in the paper, was based on the Bluespec [Flute core](https://github.com/CTSRD-CHERI/Flute).
This is a five-stage in-order pipeline that is capable of hiding the latency of some of the more complex operations.
Our production-quality core is the [CHERIoT Ibex](https://github.com/Microsoft/CHERIoT-Ibex), an area-optimised 2-3 stage pipeline.

Robert Norton-Wright and Kunyan Liu will both be present, so drop by the poster session to chat and attend their talk in session 5A on Wednesday if you're there!

If you can't make it, the paper and poster are both online (see below).
This paper focuses on the hardware.
If you would like to understand the software stack more, we've been gradually writing [documentation for the RTOS](https://github.com/microsoft/cheriot-rtos/tree/main/docs) and aim to add more soon.

Full citation
-------------

{% bibliography --cited %}

