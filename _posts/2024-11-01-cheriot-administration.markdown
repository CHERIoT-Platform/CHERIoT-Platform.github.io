---
layout: post
title:  "Who controls the CHERIoT project? (or: CHERIoT is not WordPress)"
date:   2024-11-01
categories: govenance organisation
author: "David Chisnall"
---

If you're a keen follower of open-source drama, you'll have seen that a disagreement between the maintainer of the WordPress open-source project and one of the large WordPress hosting services has spilled over onto users.
This may have made you nervous about depending on open-source projects.
I wanted to take some time to explain why the WordPress situation should not happen here.

# A project is its community

We started the project that would become CHERIoT at Microsoft Research around five years ago (it didn't have a name then, and you can see this in some of the older tests that simply say CHERI MCU in the name).
Even before it was open sourced, it was a collaborative effort with major contributions from several people on the team.
When we published the MICRO '23 paper, we listed five authors without whom the project would definitely have failed.
Today, there are even more people who can look at the system and see their fingerprints all over some of the key places in the design and implementation.

We open sourced it in early 2023 to encourage broader collaboration.
Microsoft had no interest in maintain a proprietary RISC-V extension and associated software stack but did see a benefit in a secure microcontroller ecosystem existing.
This is one of the key economic benefits of open source: no single company (or person) needs to spend the money to build and maintain a complete system, everyone can benefit from the contributions of everyone else.

We put it on GitHub, because that's the lowest-friction way for most people to communicate, but we've tried to avoid people needing to sign up to any proprietary service to collaborate with us.
I realise that some people object to GitHub's conditions of service, but my experience with running a GNU project is that the alternatives to GitHub sadly exclude more people than GitHub.

GitHub supports anonymous clones, so you don't need an account to access the code.
The project's real-time chat is done via Signal, which has a very friendly [privacy policy](https://signal.org/legal/) that we hope no one would object to.
The linked page provides this summary:

> **Privacy of user data**. Signal does not sell, rent or monetize your personal data or content in any way – ever.

If you're happy with that, you can come and ask us questions without needing a GitHub account.

We've also worked hard to make it easy for people to try CHERIoT.
If you can use Docker or Podman, you can run our dev container image on x86-64 or AArch64 platforms (you can probably build it on other architectures) and if you use VS Code or some other dev-container-aware editor then you can just open the repository and use the dev container automatically.
If you can't use these tools, we've written up instructions for building all of the dependencies by hand.
We have some people working on FreeBSD and some on PowerPC Linux, for example, so we're trying not to exclude people who don't use the big three platforms.

Making the project easy to use and easy to get involved with is very important to me personally and it's had some amazing benefits.
We've seen folks at Oxford and RPTU formally verify properties of the CHERIoT Ibex core.
We've seen folks at a variety of companies and universities build exciting things on top of the platform.
We've seen contributions across the hardware and software stack from many different people.
We've seen lowRISC build an amazing [FPGA prototyping board tailored for CHERIoT](https://www.mouser.co.uk/new/newae-technology/newae-sonata-one-dev-board/).
Yesterday, I was at the Digital Catapult CHERI Technology Access Programme Cohort 6 launch event, where participants can build on either CHERIoT or Arm's Morello and *all* of the participants in this cohort are using CHERIoT.

Back in July, Microsoft [moved the core CHERIoT projects to the CHERIoT-Platform organisation on GitHub](rtos/sail/2024/07/31/moving-to-the-cheriot-org.html) to make it easier for the CHERIoT project to exist as an independent entity.

An open-source project is driven by its contributors, but that doesn't just mean the people who write the code.
It means the people who try it and give feedback on improvements to our ISA and APIs.
It means the people who find bugs and send reduced test cases that let us fix issues.
It means the people who point at confusing bits of documentation that let us make life easier for the next person who tries the platform.
All of these people make the project better for everyone.

I strongly believe that the people in a project are more important than any governance structure.

# Who can press the emergency-stop button?

All of that said, purely pragmatically, there have to be some people in control over a project's infrastructure.
For us, that primarily means the GitHub project.
The CHERIoT GitHub organisation has three people with the owner role:

 - David Chisnall (me), SCI Semiconductor.
 - Yucong Tao, Microsoft.
 - Ben Laurie, Google.

If I ever decide to do to CHERIoT what Matt Mullenweg did to WordPress, I strongly suspect that Google and Microsoft would object.

That's not to say that I don't have a commercial interest in CHERIoT.
SCI Semiconductor announced last week that we will be [shipping the first devices in our ICENI family of CHERIoT microcontrollers next year](https://www.scisemi.com/news-1/press-release-iceni-family/).
I don't expect us to be the only people shipping CHERIoT devices and the ecosystem benefits from second sources.
The microcontroller market is tens of billions of devices each year.
I would love to see 100% of those become CHERIoT devices, but they won't all be SCI ICENI parts.

Beyond the GitHub project, I am one of five admins in the Signal chat and am the owner of the cheriot.org domain.
Given that cheriot.org already mostly contains my ramblings, I probably can't do much damage with that.
Microsoft owns cheriot.com, which currently just redirects to cheriot.org, but could point somewhere else if I decide to do something bad with cheriot.org.

# What about a CHERIoT Foundation?

CHERIoT is still a very young open-source project (it hasn't even been open source for two complete years yet).
As such, our need for bureaucracy is low.
We are mostly able to exist with free hosting and CI, and contributors are either volunteers or paid by their employers to work on the project.
We don't do anything yet that needs us to be able to take money to maintain the project.

Having a foundation would not currently provide us with any tangible benefits and would incur a lot of overhead.
I don't want to create the kind of pay-per-play structure that excludes individual contributors and demands money from commercial vendors.

As the project grows, we may need a legal non-profit entity to be the legal home.
The CHERIoT project is set up so that we can transfer control to a Foundation easily if this is required.
We won't do that until it's necessary though, and won't adopt any governance structure without consensus from our community of amazing contributors.
