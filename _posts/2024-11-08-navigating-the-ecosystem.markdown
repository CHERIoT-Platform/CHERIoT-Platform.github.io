---
layout: post
title:  "Finding your way around the CHERIoT ecosystem"
date:   2024-11-08
categories: rtos sonata git
author: Phil Day
---
With a new set of folks in the DSbD cohort 6 just getting started with their Sonata boards I thought it might be useful to capture some of the things I've learn in the last few months about the ecosystem.

### Setting up a dev environment
The easiest and quickest way to get up and running is to use the dev container.
I have a fairly strong aversion to installing new software on my base system, and this just requires that you have docker installed.
Opening a repo with dev container support in VS code will ask you if you want to reopen in a container.
Do that and in a few seconds you're in a pre-built dev environment.
Simples, to paraphrase a meerkat.

One word of warning though; cheriot and Sonata both use the ":latest" tag as the version of the docker image, so you won't automatically get updates when they are released.
If you've pulled an update to a repo and builds starts failing with no apparent reason it's worth pulling the latest dev container image and relaunching the dev container.
You can find the image name to pull in the .devcontainer/devcontainer.json file.

### Which Repo do I use ?

The CHERIoT ecosystem is split across a number of git repositories, and makes extensive used of [submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules) to link them and other external code from other places such as FreeRTOS.
There is no version or release tagging in the repos, so you can only tell how current a submodule is by comparing the commit hashes.
Versioning will start with the CHERIoT 1.0 release, which you can track [here](https://github.com/orgs/CHERIoT-Platform/projects/2).
On the plus side the CI system and maintainers do a good job of keeping it all compatible, and any breaking changes are discussed in the Signal channel.

#### [cheriot-rtos](https://github.com/CHERIoT-Platform/cheriot-rtos.git)
Cheriot-rtos is the core of the system, an includes its own set of examples that provide a great introduction to the main CHERI features.
You don't even need any hardware to run them, as by default they build for one of the two emulator environments (sail and ibex-safe-simulator) that are included in the dev container.
The ReadMe file in the examples directory has a good description of what each does and how to build the and run them.
You can also build and run the examples on the Sonata board, but remember to also set up the terminal emulator in another window or you won't see any output.

#### [cheriot-network](https://github.com/CHERIoT-Platform/network-stack.git)
This contains the code for the various compartments in the secure [network stack](https://cheriot.org/_rtos/networking/auditing/2024/03/08/cheriot-network-stack.html), and includes examples for SNTP, HTTP, HTTPS, and MQTT using some public servers.
Note though that this is really set up to be used as a submodule; there is no dev container support in here, and the examples will only build if this is cloned alongside cheriot-rtos.

It's worth looking at the layout of the code in here and how it is pulled into the other examples as an template of how to create reusable compartments.

#### [cheriot-demos](https://github.com/CHERIoT-Platform/cheriot-demos.git)
This contains examples that are more complete systems than the feature focused examples in cheriot-rtos.
The current examples are:

- _HughTheLightbulb_ requires a Sonata board connected via ethernet to network with a DHCP server and access to pool.ntp.org and test.mosquitto.org.
It then controls the RGB LEDs on the Sonata Board via MQTT and shows the CPU and heap usage on the LCD.
Its main focus is on the fault isolation within the [network stack](https://cheriot.org/_rtos/networking/auditing/2024/03/08/cheriot-network-stack.html), but it's also a useful example of basic IO programming on the Sonata board. 

- _Compartmentalisation_ runs on Sonata or Arty A7 boards, and shows how compartments can be used to create an isolation boundary around a JavaScript runtime.
It also shows how to expose the MMIO operations to JS.
There is also a version of this in cheriot-rtos/exercises that runs just in the ibex-safe-simulator. 

- _Configuration-broker_ shows how the CHERIoT features can be used to create a system where configuration data can be received from an external source and securely and safely distributed to a number compartments with a simple and minimal trust model. 
It currently builds for the ibex-safe-simulator with a simulated set of inputs and configuration targets, but a Sonata based variant also using MQTT is coming soon.   

It includes cheriot-rtos and cheriot-network as submodules, so you can also run all of the examples in those repos from here as well.
The commits of the submodule lags a little bit behind the main repos, but is generally recent enough unless you want to live on the bleeding edge.
I think this is a better place for most folks to start, especially if you want at some point to use the network stack.
You can still run all of the examples, and buffered a bit from changes in rtos, have access to some basic examples of working Sonata, and if you create an interesting demo you're all set to contribute it back.   

#### [sonata-software](https://github.com/lowRISC/sonata-software)
If you want examples that provide a wider coverage of interacting with the Sonata hardware then this is the one to look at.
It also has a dev container, but it's different from the Cheriot-rtos one and it's not clear if it is really supported or not.
Lowrisc instead suggest using [nix](https://nixos.org/), and the startup guide in the repo will talk you though installing nix on your system first.
The dev container seems to also provides a nix setup, although it's not the easiest thing to use.
I found for example that the default (nix-devshell) terminal wouldn't launch and I had to select bash terminal instead, and within that run everything as root. 
This also has a submodule of cheriot-rtos, but it's a fork that has a few Sonata changes waiting to be upstreamed.

### Connecting up a Sonata Board
[From zero to CHERIoT in two minutes with Sonata](https://cheriot.org/fpga/ibex/2024/06/10/sonata-quick-start.html) covers how to extend a dev container to include a Sonata board.

If you power up your Sonata board after opening VS Code, then you need to tell VS code to rebuild the dev container (Ctrl-Shift-P -> Dev Containers:Rebuild Container) to get it to re-establish the volume mount. 

If you're going to be running any of the cheriot-rtos examples with console output then you also need to run a terminal emulator such as [picocom](https://linux.die.net/man/8/picocom) or [putty](https://www.putty.org/) on whichever tty / COM port the Sonata board is connected to. 




