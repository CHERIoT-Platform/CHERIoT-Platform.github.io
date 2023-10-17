---
layout: post
title:  "Ibex simulator now available in the devcontainer"
date:   2023-10-17 11:38:13 +0100
categories: jekyll update
---

Since its creation, the CHERIoT DevContainer has included the simulator built from the Sail formal model.
This is an instruction-accurate simulator and is the gold model for the CHERIoT ISA.

With the work from Microsoft on providing a [complete FPGA simulation environment for the CHERIoT Ibex](https://github.com/microsoft/cheriot-safe), it's now possible to build a complete simulation.
If you have an Arty A7, you should be able to build this platform and run at a realistic speed.

For the rest of us, there's [verilator](https://verilator.org).
Verilator builds a software simulation from the verilog.
This is now built in two configurations in the dev container:

 - `cheriot_ibex_safe_sim` runs the simulation.
 - `cheriot_ibex_safe_sim_trace` runs the simulation and writes a file with per-instruction tracing.

These expect two files containing hex dumps of memory in the `firmware` directory (a subdirectory of the working directory).
The first, `cpu0_irom.vhx` contains ROM bootloader code.
It's sufficient for this to contain a single jump instruction that branches to `cpu0_iram.vhx`.

Creating these is not completely trivial and so we've included [a script to run with the simulator](https://github.com/microsoft/cheriot-rtos/blob/main/scripts/run-ibex-safe-sim.sh).
This script, in turn, is invoked by `xmake run`.
If you're in the dev container, it will find all of the tools that it needs automatically.

This means that you can now build and run the examples in a simulation that behaves like real hardware (only much slower):

```
$ cd cheriot-rtos/examples/01.hello_world/
$ xmake config --sdk=/cheriot-tools/ --board=ibex-safe-simulator
checking for platform ... cheriot
checking for architecture ... cheriot
generating /home/cheriot/cheriot-rtos/sdk/firmware.ldscript.in ... ok
$ xmake 
[ 23%]: cache compiling.release ../../sdk/lib/atomic/atomic2.cc
...
[ 95%]: linking firmware build/cheriot/cheriot/release/hello_world
[ 95%]: Creating firmware report build/cheriot/cheriot/release/hello_world.json
[ 95%]: Creating firmware dump build/cheriot/cheriot/release/hello_world.dump
[100%]: build ok, spent 1.58s
warning: /home/cheriot/cheriot-rtos/sdk/xmake.lua:102: unknown language value 'c2x', it may be 'c89'
warning: add -v for getting more warnings ..
$ xmake run
Reading firmware/cpu0_irom.vhx
%Warning: firmware/cpu0_irom.vhx:33: $readmem file ended before specified final address (IEEE 2017 21.4)
%Warning: firmware/cpu0_iram.vhx:9449: $readmem file ended before specified final address (IEEE 2017 21.4)
Hello world compartment: Hello world
Error handler: Thread exit CSP=0x20040930 (v:1 0x20040730-0x20040930 l:0x200 o:0x0 p: - RWcgml -- ---)
swci_main exiting with return code 00
```

If something goes wrong, you can try rerunning with `cheriot_ibex_safe_sim_trace`:

```
$ cd build/cheriot/cheriot/release
$ $ /cheriot-tools/bin/cheriot_ibex_safe_sim_trace 
Reading firmware/cpu0_irom.vhx
%Warning: firmware/cpu0_irom.vhx:33: $readmem file ended before specified final address (IEEE 2017 21.4)
%Warning: firmware/cpu0_iram.vhx:9449: $readmem file ended before specified final address (IEEE 2017 21.4)
TOP.swci_vtb.dut.msftDvIp_cheri0_subsystem_i.msftDvIp_cheri_core0_i.msftDvIp_cheri_core_wrapper_i.ibex_top_i.u_ibex_tracer.printbuffer_dumpline.unnamedblk1: Writing execution trace to trace_core_00000000.log
Hello world compartment: Hello world
Error handler: Thread exit CSP=0x20040930 (v:1 0x20040730-0x20040930 l:0x200 o:0x0 p: - RWcgml -- ---)
swci_main exiting with return code 00
$ ls -lah trace_core_00000000.log 
-rw-r--r--. 1 cheriot cheriot 13M Oct 17 13:11 trace_core_00000000.log
```

Note that these trace files can get quite large: 13 MiB from the hello world example!
