---
layout: post
title:  "Clang 17 now in dev container, and other toolchain news"
date:   2025-01-23
categories: cheri toolchain
author: Owen Anderson
---

Thanks to a collaboration with the [upstream CHERI toolchain](https://github.com/ctsrd-cheri/llvm-project), the CHERIoT toolchain has now been rebased onto Clang 17 from Clang 13, bringing us two years closer to upstream Clang.
Major thanks to [Alex Richardson](https://github.com/arichardson) (Google) and [Sam Leffler](https://github.com/sleffler/) (Google) for their work on this effort!

This update brings with it substantially improved support for C++20 features, as well as preliminary support for some C++23 and C23 features.
It also brings with it many improvements to the core of the RISC-V code generator, notably benefiting code size: the firmware image for the Cheriot RTOS testsuite is 2.7% smaller when built with Clang 17 compared to Clang 13.
Other highlights include compile time improvements, and too many under-the-hood fixes to enumerate.
You can find detailed release notes for all Clang and LLVM releases on the LLVM.org [releases page](https://releases.llvm.org).

While we generally attempt to maintain compatibility between CHERIoT RTOS and the older toolchain, we recommend pulling the latest devcontainer (or otherwise updating your toolchain) to ensure the best experience.

## Other Toolchain Improvements

Since landing the Clang 17 rebase, we've been busy bringing bugfixes and enhancements to the CHERIoT toolchain, including:

### Language & Usability Improvements

- [Implemented](https://github.com/CHERIoT-Platform/llvm-project/commit/025c5d452e8935ebbe2a09d78fb2a10c1c96a626) a new Clang diagnostic to warn on compartment exports that return void, or where the return value is unused.
This is important in practice, because cross-compartment calls can fail in the compartment switcher, and void returns make this failure undetectable.
These warnings are disabled by default until CHERIoT RTOS has been updated for them, and are controlled by the `-Wcheri-compartment-return-void` compiler flag. Thanks to [Robert Dazi](https://github.com/v01dXYZ) for this one!

- [Allowed](https://github.com/CHERIoT-Platform/llvm-project/commit/0de0fb3e8f63be9102c5b5eab1b496415b667ca9) `cheri_libcall`-annotated functions to decay into unannotated function pointers.
This is useful for passing the address of a `cheri_libcall` function as a callback within a compartment.

- [Improved](https://github.com/CHERIoT-Platform/llvm-project/commit/b14e86345d929bf91ab3fb1197ac716dc7ca6e2d) linker error reporting if you accidentally omit the compartment export annotation on a declaration.
`lld` will now look for matching unexported functions and provide a suggested fix.

- [Eliminated](https://github.com/CHERIoT-Platform/llvm-project/issues/58) the need to repeat the minimum stack size in both the annotation and in the stack check, improving the ergonomics significantly.
Below is an example of using the `StackUsageCheck` template in CHERIoT RTOS to verify stack usage, demonstrating the older style that repeats the size, and the new style that does not.
The related `STACK_CHECK(expected)` macro in CHERIoT RTOS has been updated to use the new style, and the `expected` parameter will be removed in the future.
```c++
    __cheriot_minimum_stack(0x200)
    int old_style() {
        StackUsageCheck<StackCheckMode::Asserting, 0x200, __PRETTY_FUNCTION__> stackCheck;
    }

    __cheriot_minimum_stack(0x200)
    int new_style() {
        StackUsageCheck<StackCheckMode::Asserting, __cheriot_minimum_stack__, __PRETTY_FUNCTION__> stackCheck;
    }
```

- [Added support](https://github.com/CHERIoT-Platform/llvm-project/issues/38) for "temporal" capability valid bit checking, using a new builtin `__builtin_cheri_tag_get_temporal(void*)`.
This is needed when reading the valid bit in situations where validity can change within the current function, such as around a deallocation or pinning.
In all other circumstances, the existing non-temporal version should be preferred for better optimization.
An example would be using a double-checked pattern when pinning with `heap_claim_fast`, where combining the two valid bit reads would yield incorrect code.
```c++
void func(Timeout *t, int *ptr) {
    if (__builtin_cheri_tag_get_temporal(ptr)) {
        int claim = heap_claim_fast(t, ptr, nullptr);
        if (claim == 0 && __builtin_cheri_tag_get_temporal(ptr)) {
            *ptr = 1234;
            // ...
        }
    }
}
```

### Bugfixes

- [Fixed](https://github.com/CHERIoT-Platform/llvm-project/commit/60b4a582dfc1579b3c08c65d4b6ede961eb267f5) a recurring issue where the compiler would generate improperly mangled calls to `memcpy`, `memmove` and/or `memcmp` in specific circumstances, resulting in linker errors. 
This has now been fixed at the source.
- [Fixed](https://github.com/CHERIoT-Platform/llvm-project/issues/57) linker errors that arise when taking the address of non-exported, non-libcall functions with non-default interrupt state annotations.

### Optimizations
- [Taught](https://github.com/CHERIoT-Platform/llvm-project/commit/25ad11d7832237e81ca476d4e3e6bac2defc3fa7) the compiler to better optimize `CAndPerm` instructions, including constant folding and idempotence.
This tends to benefit places where redundant `CAndPerm` instructions were generated by macros or C++ templates.
- [Freed](https://github.com/CHERIoT-Platform/llvm-project/commit/8221b74cffbfa03149eb5bab1776280ebb43785f) up the TP/X4 register for the compiler's use in code generation.
This register is normally reserved as a "thread pointer" in RISC-V ABIs, but is not used for that purpose on CHERIoT.
We haven't observed this making a significant performance or size difference, but some compute-intensive code may benefit.
- [Re-enabled the MachineOutliner](https://github.com/CHERIoT-Platform/llvm-project/issues/46) size optimization. This  improved the firmware size on the CHERIoT RTOS testsuite by 4.4%, and will likely benefit other code bases similarly.
However, this optimization uncovered an issue in the CHERIoT ISA related to return sentinels that has since been [fixed in the specification](https://github.com/CHERIoT-Platform/cheriot-sail/issues/85).
If your development board does not contain the fix, you will need to pass `-enable-machine-outliner=never` to the compiler.
We have added automatic support for enabling this flag when required to the CHERIoT RTOS build system prior to enabling the optimization by default.

## Looking Forward

We have a number of further improvements that we expect to make available to CHERIoT toolchain users in the near future:

- [Supporting sealed capabilities](https://github.com/CHERIoT-Platform/llvm-project/pull/88) in the C type system.
This change adds a new pointer attribute `__sealed_capability`, which disallows any operations that would cause the pointer to be dereferenced, or to be lose its `__sealed_capability` annotation.
Once integrated with CHERIoT RTOS, we will be able to represent sealing, unsealing, and the propagation of sealed capabilities in a type-safe manner.
This is demonstrated below.
```c++
// CHERI sealing and unsealing operations now have signatures that are type-safe with respect to sealing.
void * __sealed_capability cheri_seal(void *cap, const void *type);
void * cheri_unseal(void * __sealed_capability cap, void *type);

int func(int * __sealed_capability ptr) {
    // This causes a compiler error!
    return *p;
}
```
- [Improving code quality](https://github.com/CHERIoT-Platform/llvm-project/issues/85) for unaligned memory accesses.
We've seen this particularly benefitting some cryptographic code.
