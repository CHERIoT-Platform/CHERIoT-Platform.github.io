---
layout: post
title:  "My TAP Journey"
date:   2025-03-17
categories: tap sonata cheriot-rtos
author: Adam Finney
---

Over the past few months, I have had the opportunity to work with CHERIoT and Sonata in ways that have really pushed me as a developer.
When I started this project, I knew that CHERI's capability based memory model was something special, but I did not fully appreciate how much it would change the way I think about secure embedded systems.
This has been more than just a technical challenge. It has been a genuine learning experience that has reshaped how I approach programming, security, and system design.

# Discovering CHERIoT and Sonata

When I first started working with CHERIoT, I knew it was designed to tackle one of the biggest problems in embedded development: security, buffer overflows, memory corruption, and other memory related issues have plagued embedded systems for decades.
CHERIoT's capability based memory model offers a fundamentally different way of handling these problems at the hardware level.
Instead of patching the symptoms, it removes the underlying vulnerabilities.

Sonata, as the development platform for CHERIoT, gave me the chance to put these ideas into practice.
It was a bit daunting at first. Figuring out how to structure code within that framework required a real shift in mindset.
But once it started to click, I began to see how powerful it could be.
Instead of constantly worrying about memory safety and unexpected crashes,
I could focus more on building functionality, knowing that the hardware was helping to protect me from some of the most common programming mistakes.

# Pushing Through Challenges

There were definitely some challenging moments along the way. Early on, I tried to implement lwIP for networking, but it became clear that it was not the right fit.
After seeing the [Hugh the Lightbulb](https://github.com/cheriot-Platform/cheriot-demos) demo, I switched to FreeRTOS plus TCP, and that turned out to be the right move.
FreeRTOS plus TCP integrated much more smoothly with CHERIoT.

Networking was a big focus for me.
I have been working on finalising the IPv6 and UDP stack and preparing it for open source release.
The packet sniffing and IPv6 proof of concept has been open source for a while now, but I realised it was using more power than it should because the filters were completely pass-through.
Fixing that turned out to be more complicated than I expected.

IPv6 and DTLS add an extra layer of complexity that required more than just tweaking the code.
This was solved with the help of the community with compiler optimisations and the integration of HyperRAM.
At the time of writing, the stack is in the shake down phase.
That frustrating but exciting period where you know you are close to the finish line, but the last ten percent of the work feels like half the effort.

# Shifting My Approach to Code

One of the biggest changes for me has been how I now think about code structure.
Before CHERIoT, I would organise code by function, grouping similar tasks together to keep things tidy and efficient.
But CHERIoT's memory model encourages a different way of thinking.

I started organising code by safety boundaries rather than function.
I separated input and output from parsing and business logic, setting up compartments where failures in one area could not compromise the whole system.
It required more upfront design work, but the payoff was huge.
Debugging became easier, failures were contained, and the overall stability of the system improved significantly.

Compartmentalisation has changed the way I write code, not just on CHERIoT but across other projects as well.
It makes you more thoughtful about how data flows through a system and where the vulnerabilities might be hiding.

# What I Have Learned

I think the biggest lesson I have taken away from this experience is that the hardest problems often need a completely new way of thinking.
Trying to fix memory safety at the software level will only take you so far.
You need to solve it at the hardware level, which is exactly what CHERIoT does.

Working with a strong community makes a huge difference.
Being able to ask questions, share ideas, and learn from others' experiences has made the process so much smoother.
The idea of combining a top down strategy with a bottom up, grassroots approach feels like the right way to drive adoption and make CHERIoT and Sonata a success.


