---
layout: post
title:  "Clang 20 is now in the CHERIoT devcontainer!"
date:   2025-04-23
categories: cheri toolchain
author: Owen Anderson
---

As of today, the [Clang 20](https://releases.llvm.org/20.1.0/tools/clang/docs/ReleaseNotes.html)/[LLVM 20](https://releases.llvm.org/20.1.0/docs/ReleaseNotes.html) toolchain is available in the CHERIoT devcontainer.
This follows on the heels of our recent releases of [Clang 18](https://releases.llvm.org/18.1.0/tools/clang/docs/ReleaseNotes.html)/[LLVM 18](https://releases.llvm.org/18.1.0/docs/ReleaseNotes.html) and [Clang 19](https://releases.llvm.org/19.1.0/tools/clang/docs/ReleaseNotes.html)/[LLVM 19](https://releases.llvm.org/19.1.0/docs/ReleaseNotes.html) in recent months.

With this release, we are now caught up to the most recent upstream Clang release.Going forward, we will be continuously tracking upstream Clang development in a `cheriot-upstream` branch within the [toolchain repository](https://github.com/cherIoT-Platform/llvm-project), and we will release new major versions to CHERIoT developers as they are released upstream as well.

As we move beyond toolchain updates, we will be refocusing toolchain development on further enhancements, such as:

* Adding support for [LLDB](https://lldb.llvm.org) as a debugger for CHERIoT hardware.
* Implementing toolchain primitives needed to unblock [Rust](https://www.rust-lang.org) on CHERIoT, including burning down inline assembly [hacks](https://github.com/CHERIoT-Platform/llvm-project/issues/6).
* Preparing the toolchain for ABI freeze, which will guarantee binary compatibility going forward.
* Investigating performance and code size gaps compared to RISC-V.
* User-requested bug fixes and improvements.

As always, don't hesitate to report any issues or areas for improvement on the CHERIoT toolchain [issues page](https://github.com/CHERIoT-Platform/llvm-project/issues)!