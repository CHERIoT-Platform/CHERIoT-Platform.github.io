# MPACT-Cheriot

## A fast and flexible simulator for the CHERIoT architecture.

### Tor Jeremiassen, Google LLC.

## Introduction

The Cybersecurity and Infrastructure Security Agency (CISA) has identified
memory safety vulnerabilities as a major cybersecurity risk \[1\], pointing to
reports that 70% or more of security vulnerabilities were found to involve
memory-safety issues. CHERIoT \[2\] is a new architecture that seeks to provide
strong protection against many frequently exploited memory vulnerabilities.
CHERIoT is based on using CHERI \[3\] capability hardware-extensions to a 32 bit
RISC-V \[4\] platform to provide fine-grained spatial and temporal safety, along
with lightweight software compartmentalization. Standards like these are
critical to broad adoption of security features \[5\], but companies
implementing these also need a firm understanding of the performance tradeoffs
of their implementations. In this article I introduce MPACT-Cheriot \[6\], an
instruction-set simulator (ISS) that is designed to be extensible, easily
modifiable, and fast, in order to enable rapid prototyping and debugging of the
new architecture. MPACT-Cheriot is built on top of MPACT-RiscV \[7\], a RISC-V
ISS built using the MPACT-Sim framework \[8\], all developed at Google and
available on Github.

## MPACT-Sim

MPACT-Sim is an ISS framework in C++ that was designed with the goal to improve
the velocity of the simulator developer, to make it easy to create ISSs from
scratch, and to rapidly and easily modify existing ISSs in response to ISA
design changes or user-needed functional enhancements. In other words, MPACT-Sim
enables rapid HW/SW co-design and early pre-Silicon software development.

Manually writing an instruction decoder is tedious and error prone. MPACT-Sim
removes this task by providing tools that read in instruction descriptions and
generate C++ code for instruction decoding and bit field extractions. This
allows the developer to avoid writing thousands of lines of code and instead
focus on instruction semantics and simulator functionality. The tools also
generate code for instruction encoders to accelerate writing of assemblers,
which is particularly useful in early stages of ISA development/extension, when
regular codegen tools either do not exist for the architecture or have not been
updated to use newly added instructions. It also avoids the need to be an expert
in LLVM/GCC to iterate quickly on new instructions.

```
    format Inst32Format[32] {
        fields:
            unsigned bits[25];
            unsigned opcode[7];
    };
    
    format IType[32] : Inst32Format {
        fields:
            signed imm12[12];
            unsigned rs1[5];
            unsigned func3[3];
            unsigned rd[5];
            unsigned opcode[7];
      overlays:
            unsigned u_imm12[12] = imm12;
            unsigned i_uimm5[5] = rs1;
    };

    instruction group RiscVGInst32[32] : Inst32Format {

        addi   : IType  : func3 == 0b000, opcode == 0b001'0011;

    };
```
Figure 1. Encoding specification for RISC-V `addi` instruction.

The instruction descriptions are divided into two pieces, each containing
multiple pieces of information about each instruction. The first piece focuses
on the encoding, describing the instructions' encoding formats, the size, and
detailed encoding, as seen in Figure 1\. This description also allows for the
description of *overlays,* which allows bitfields to be reinterpreted as
different types, broken up, and concatenated (including with constants) to
create new *virtual* *bitfields*. One use for this is to describe the
RISC-V/CHERIoT instructions whose immediate fields are non-contiguous.

```
    addi{: rs1, %reloc(I_imm12) : rd},
        resources: {next_pc, rs1 : rd},
        disasm: "addi", "%rd, %rs1, %I_imm12",
        semfunc: "&RV64::RiscVIAdd";
```
Figure 2. Instruction specification for RISC-V `addi` instruction

The second piece of the descriptions (seen in Figure 2.) specify the
instructions' operands, their resource usage (optional), disassembly format,
and binding to the C++ callables that implement the instructions' semantics in
the simulator. The operands are enumerated in three sections delimited by
colons: a predicate operand (if any), a list of source operands, and a list of
destination operands. In Figure 2, the `addi` instruction is shown as having
source operands `rs1` and `I_imm12`, and destination operand `rd`. The
`%reloc()` decorator is used to indicate an operand that may require a
relocation entry, but is only used if an assembler is generated (see below).
Additionally, the destination operand can also be annotated with a latency that
can be used together with resource modeling to simulate the effect of non-unit
instruction latencies.

Next, you can optionally define resources that an instruction reads and writes
to make it easier to implement different instruction "issue" strategies that
allow you to take into account the dependencies and the latencies between
instructions.

A disassembly formatting string can be specified. This is used to create a
representation of each decoded instruction, making it easy to create
disassemblers and human readable traces of instructions, as well as allowing
instructions to be printed in a command-line debug interface.

Lastly, the description includes a binding to the C++ callable in the simulator
source that will be used to implement the instruction's semantics and called
every time the instruction is executed.

Loops present an opportunity to improve simulation speed. Therefore, when an
instruction is decoded, the information about the instruction is stored in an
instruction descriptor. This descriptor is stored in an internal
*instruction cache* that eliminates the need to re-decode previously executed
instructions, i.e., instructions from previous iterations of a loop. The cached
instruction descriptors are invalidated individually, by range, or as a whole in
response to updates of the instruction memory that could modify instruction
words. This support for using software breakpoint instructions, overlays,
run-time code-generation, and self-modifying code.

| ![][image1] |
| :---- |
| Figure 3. Possible instruction-descriptor hierarchy tree for a VLIW instruction with two bundles, each consisting of 3 instructions, with two instructions broken into two phases. |

More advanced features allow instructions to be grouped in bundles (VLIW
instruction words) and hierarchies of bundles, and for instruction semantics to
be divided into multiple phases with separate instruction descriptors for each
phase. An example of the latter allows the write-back of a load instruction to
be executed separately (and asynchronously) from the address generation and
memory request.

Each instruction semantic-function callable is invoked when the `Execute()`
method of an instruction-descriptor object is called. The semantic-function
callable takes a pointer to an instruction descriptor as its parameter. The
instruction descriptor contains vectors of pointers to access the interfaces of
the instruction operands. These interfaces are used to avoid having to
differentiate between accessing different operand types, such as literals,
immediate values, and register values. This allows the same semantic function
to be used for multiple instructions if they differ only in the type of their
operands, e.g., add register and add immediate.

MPACT-Sim provides classes to support modeling a large set of different
architecture state objects. There are templated classes for use in modeling
registers with different value types, as well as modeling scalar registers,
vector registers, and array registers. The methods in the operand-access
interfaces support accessing values from any of these objects.

```c++
    // Generic helper function for binary instructions.
    template <typename Register, typename Result, typename Argument>
    inline void RiscVBinaryOp(const Instruction *instruction,
                              std::function<Result(Argument, Argument)> operation) {
        using RegValue = typename Register::ValueType;
        Argument lhs = generic::GetInstructionSource<Argument>(instruction, 0);
        Argument rhs = generic::GetInstructionSource<Argument>(instruction, 1);
        Result dest_value = operation(lhs, rhs);
        auto *reg = static_cast<generic::RegisterDestinationOperand<RegValue> *>(
            instruction->Destination(0))->GetRegister();
        reg->data_buffer()->template Set<Result>(0, dest_value);
    }
    // Instruction semantic function for 'add' and 'addi'.
    void RiscVIAdd(const Instruction *instruction) {
        RiscVBinaryOp<RegisterType, UIntReg, UIntReg>(
                instruction, [](UIntReg a, UIntReg b) { return a + b; });
    }
```
Figure 4. The semantic function used for RISC-V `add` and `addi` instructions.

Unlike many other simulators and infrastructures, registers in MPACT-Sim are
named and not numbered. That is, a register is identified by its name, not an
index into a register-file structure implied by the instruction opcode (e.g.,
integer vs floating point). All registers derive from a common base class, and a
map container is used to map from the name to the register object. The register
name is generated by the decoder, which then uses it to look up the register
object. This provides an abstraction from the register architecture of the
target CPU and makes it easier to perform experiments with changing the number
and organization of registers without impacting the simulator outside of the
decoder.

Instruction semantic-functions access source and destination operands through
abstract interfaces. The source-operand interface is shown below in Figure 5:

The interface is implemented by different operand-access objects customized to
each underlying object. For instance, there is a register-access operand class
as well as one for immediate values. This indirect method of accessing register
operands does add some run-time overhead, but provides a great deal of
flexibility. Instruction semantic functions are agnostic not only about the
specific registers they work on, but also on whether the operand is a register
or an immediate value. This indirection avoids coupling between the instruction
semantics, operand types, and register architecture, allowing one to make
significant changes in the register architecture and operands without modifying
the instruction semantic functions.

```c++
    // The source operand interface provides an interface to access input values
    // to instructions in a way that is agnostic about the underlying implementation
    // of those values (eg., register, fifo, immediate, predicate, etc).
    class SourceOperandInterface {
      public:
        // Methods for accessing the nth value element.
        virtual bool AsBool(int index) = 0;
        virtual int8_t AsInt8(int index) = 0;
        virtual uint8_t AsUint8(int index) = 0;
        virtual int16_t AsInt16(int index) = 0;
        virtual uint16_t AsUint16(int) = 0;
        virtual int32_t AsInt32(int index) = 0;
        virtual uint32_t AsUint32(int index) = 0;
        virtual int64_t AsInt64(int index) = 0;
        virtual uint64_t AsUint64(int index) = 0;
        // Return a pointer to the object instance that implements the state in
        // question (or nullptr if no such object "makes sense"). This is used if
        // the object requires additional manipulation - such as a fifo that needs
        // to be popped. If no such manipulation is required, nullptr should be
        // returned.
        virtual std::any GetObject() const = 0;
        // Return the shape of the operand (the number of elements in each dimension).
        // For instance {1} indicates a scalar quantity, whereas {128} indicates an
        // 128 element vector quantity.
        virtual std::vector<int> shape() const = 0;
        // Return a string representation of the operand suitable for display in
        // disassembly.
        virtual std::string AsString() const = 0;
        virtual ~SourceOperandInterface() = default;
    };
```
Figure 5. Source-operand access interface used in instruction semantic
functions.

## MPACT-Cheriot

MPACT-Cheriot provides an ISS for the CHERIoT architecture using the MPACT-Sim
infrastructure, similar to and based on the RISC-V simulators provided in
MPACT-RiscV. It reuses many of the instruction semantic functions and
instruction descriptions of the RISC-V 32 bit instruction set, and adds or
modifies instruction descriptions for the CHERIoT instructions that access and
manipulate capabilities.

The implementation issues a single instruction per cycle, and that instruction
completes in the same cycle, so there are no dependencies to model. The
instruction execution is performed inside a *top-level* class that implements an
extension of the generic MPACT-Sim core debug-interface, allowing robust debug
capability to be built on top of that class.

When invoked in *interactive* mode, the ISS is controlled from the debug
command-line interface (CLI) which uses the top-level class to provide a range
of debug commands. The ISS prompt provides the disassembly of the instruction to
be executed next, the current symbol, and additional information, such as if an
interrupt/exception was just taken or returned from. The ISS can be run,
stepped, and halted from the CLI. Registers and memory can be read and written.
Breakpoints and data watchpoints can be set and cleared. Breakpoints can be set
not only on individual instructions, but also on a change in control flow, a
taken interrupt (Figure 6\) and/or exception (Figure 7\) and its return. The ISS
detects if an interrupt or exception will immediately cause another and
immediately halts, for instance, if the first instruction in the handler would
cause an exception. The main set of registers, as well as the current
interrupt/exception stack can be shown using the the `reg $all` command as seen
in Figure 8\.

```
    [0] > break $interrupt
    [0] > run
    Stopped at taken interrupt 8000252c
    cspecialrw        csp, mtdc, csp
    interrupt Machine timer interrupt
    [0] >
```
Figure 6\. Example showing break on interrupt.

```
    [0] > break $exception
    [0] > run
    Crash recovery (inner compartment) test: Store silently ignored
    Crash recovery (main runner) test: Calling crashy compartment returned (crashes: 0)
    Crash recovery (main runner) test: Returning normally from crash test
    Crash recovery (main runner) test: Calling crashy compartment to double fault and unwind
    Crash recovery (inner compartment) test: Trying to fault and double fault in the error handler Stopped at exception 8000252c
    cspecialrw        csp, mtdc, csp exception
    Exception taken at 8000e024: CHERI exception: bounds violation c8
    [0] >
```
Figure 7\. Example showing break on exception.

```
    [0] > reg $all
    Interrupt stack:
    [0] exception  Exception taken at 80002ec0: Environment call from M-mode
    czero = 0x00000000 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    cra   = 0x80002eaa (v: 1 0x800028a0-0x080003960 l: 0x0000010c0 o: 0x4 p: G R-cmg- X- ---)
    csp   = 0x80001180 (v: 1 0x80000ad0-0x0800011d0 l: 0x000000700 o: 0x0 p: - RWcmgl -- ---)
    cgp   = 0x8001beb0 (v: 1 0x8001bd60-0x08001c000 l: 0x0000002a0 o: 0x0 p: G RWcmg- -- ---)
    ctp   = 0x00000000 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    ct0   = 0x8001bd98 (v: 1 0x8001bd80-0x08001be70 l: 0x0000000f0 o: 0x0 p: G RWcmg- -- ---)
    ct1   = 0x00000000 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    ct2   = 0x800066c0 (v: 1 0x800066a8-0x080006d00 l: 0x000000658 o: 0x1 p: G R-cmg- X- ---)
    cs0   = 0x00000001 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    cs1   = 0x8001bd80 (v: 1 0x8001bd80-0x08001be70 l: 0x0000000f0 o: 0x0 p: G RWcmg- -- ---)
    ca0   = 0x8001bf90 (v: 1 0x8001bd60-0x08001c000 l: 0x0000002a0 o: 0x0 p: G RWcmg- -- ---)
    ca1   = 0x80001240 (v: 1 0x80001240-0x080001248 l: 0x000000008 o: 0x0 p: - RWcmgl -- ---)
    ca2   = 0x00000000 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    ca3   = 0x00000000 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    ca4   = 0x00000001 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    ca5   = 0x00000000 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
    pcc   = 0x80002494 (v: 1 0x80002320-0x0800028a0 l: 0x000000580 o: 0x0 p: G R-cmg- Xa ---)
    mepcc = 0x80002ec0 (v: 1 0x800028a0-0x080003960 l: 0x0000010c0 o: 0x0 p: G R-cmg- X- ---)
    mtcc  = 0x80002494 (v: 1 0x80002320-0x0800028a0 l: 0x000000580 o: 0x0 p: G R-cmg- Xa ---)
    mtdc  = 0x80000230 (v: 1 0x80000230-0x0800003c8 l: 0x000000198 o: 0x0 p: G RWcmgl -- ---)
    mscratchc = 0x00000000 (v: 0 0x00000000-0x000000000 l: 0x000000000 o: 0x0 p: - ------ -- ---)
```
Figure 8\. Example showing the result of a `reg $all` CLI command.

The ISS keeps track of the branch history, which can be printed on demand
(Figure 9), listing the last N branch instructions and their destinations. The
branch history includes interrupts/exceptions and their returns, and summarizes
repeated branches (in a loop) by providing a branch *count*. The number of
branches captured in the branch trace can be changed in the CLI.

```
    [0] > branch-trace
        From          To              Count
    0x800037e2 -> 0x800037f6            1
    0x80003804 -> 0x80002e54            1
    0x80002e68 -> 0x8000384c            1
    0x80003886 -> 0x800066c0            1
    0x80006722 -> 0x80003888            1
    0x8000388e -> 0x80003892            1
    0x8000389e -> 0x80002e6c            1
    0x80002e6e -> 0x80002e7c            1
    0x80002e82 -> 0x80003706            1
    0x80003716 -> 0x80002e86            1
    0x80002e96 -> 0x80002e9e            1
    0x80002ea6 -> 0x800038a0            1
    0x800038ac -> 0x800038f6            1
    0x80003902 -> 0x80003912            1
    0x80003916 -> 0x80002eaa            1
    0x80002ec0 -> 0x80002494            1
    80002494   cspecialrw        csp, mtdc, csp
    [0] >
```

Figure 9\. Example showing branch history.

The MPACT-Cheriot ISS is instrumented with a number of counters, the most
important of which are for cycles and the number of instructions executed
(which, when IPC=1, are always equal), but also a counter for every opcode in
the ISA. The ISS supports cache modeling with configurable level-1 caches. These
have counters for all the basic cache statistics. All counters are exported to a
text protobuf \[9\] file at the end of simulation.

Instruction- and data-memory profiling are supported. The instruction profiler
counts the number of times each instruction located in an executable region
specified in the ELF input file is executed. The data profiler tracks which
words in the address space have been accessed, but in the interest of space (a
single bit vs. a 32 bit counter per word), does not track the access count or
type.

There are two ISS versions provided. One is a standalone CHERIoT simulator, the
other is a dynamic library that implements an interface that allows it to be
imported, instantiated, and used as a CPU in ReNode \[10\] system simulations.
ReNode is a development environment that allows you to run full unmodified
software on a model of a full system. Applications running on systems are often
distributed across a number of processors, whether on a single SoC or across
chip boundaries, and access a number of peripherals and memories across the
system. Being able to develop, run, and debug the full software pre-silicon on a
simulated platform brings with it the same benefits as developing software for a
single core using an ISS in terms of visibility, debuggability, and fan-out.

ReNode provides debugging facilities and visibility at the system level and
supports running a GDB server, but of course, only for processors already
supported by GDB. For new architectures, or newly modified architectures, GDB
support is often not available. The dynamic-library version of MPACT-Cheriot
avoids having to enable GDB support by exposing the debug CLI interface on a TCP
port, to which one can connect with telnet or a similar utility, to control the
ISS just like the standalone version.

In addition to modeling the CHERIoT CPU, the MPACT-Cheriot ISS contains models
for tagged and untagged RAM memories and memory-request routers, which are
accessed using MPACT-Sim defined interfaces (though, in general, this is up to
the implementor, as these interfaces are not mandatory, but there for
convenience). The RISC-V CLINT \[11\] (core-local interruptor) is modeled by
default (the base address is configurable). A model for the PLIC \[12\]
(platform-level interrupt controller) is available, but is not modeled by
default. In the standalone version, a simple UART (output only) is added
(configurable base address) to allow UART-based "printing".

Both simulator versions support semihosting. Semihosting allows the program
running on the simulator to interact with the host system through traps that the
simulator intercepts. Examples include functions that are normally provided by
an operating system such as host system file I/O \- including standard input,
standard output, and standard error. The semihosting follows the RISC-V standard
\[13\], which in turn is based on the ARM Semihosting Specification \[14\]. A
semihosting call is made when the following instruction sequence is encountered:

```
	slli x0, x0, 0x1f
	ebreak
	srai x0, x0, 7
```

Upon executing the `ebreak` of the semihosting call, the simulator first
verifies that the neighboring instructions (which are effectively no-ops)
conform with a semihosting call. It then examines the contents of the `a0` and
`a1` registers, which contain the semihosting-call function id and the address
of the semihosting-call parameter block, respectively. It then performs the
requested function before resuming execution of the simulated program.

The MPACT-Cheriot ISS has also been adapted to work with TestRIG \[15\], a
framework for testing RISC-V processors with random instruction generation.
TestRIG works by feeding instruction-trace streams to multiple RISC-V
implementations and comparing the result streams (encoded in a standardized
trace format) from each of the targets. This allows for easier verification of
MPACT-Cheriot against golden models, such as Sail \[16\].

In terms of performance, MPACT-Cheriot is comparatively fast at about 14 million
simulated instructions per second when executing the CHERIoT RTOS test-suite
\[17\] on a reasonably fast host. This is good performance for a simulator that
doesn't perform runtime code-generation. Moreover, this is about 48x faster
than the Sail simulator for CHERIoT for the same program and host.

In summary, MPACT-Cheriot is an open-sourced, fast simulator that has excellent
debug features to support early software development for the new CHERIoT
architecture. It is based on the MPACT-Sim framework which made it easy to
develop and modify to track ISA changes as they occurred. It can be used both
standalone and as part of a ReNode full-system simulation. Its speed and debug
features have proven invaluable internally: "What was taking hours to debug now
takes minutes\!".

## References:

\[1\] Bob Lord, "The Urgent Need for Memory Safety in Software Products", CISA blog, December 2023\. [https://www.cisa.gov/news-events/news/urgent-need-memory-safety-software-products](http://www.cisa.gov/news-events/news/urgent-need-memory-safety-software-products).

\[2\] Amar, Saar, et al. “CHERIoT: Complete Memory Safety for Embedded Devices.”
*Proceedings of the 56th IEEE/ACM International Symposium on Microarchitecture*,
2023\.

\[3\] Watson, Robert N. M., et al. “An Introduction to CHERI.” *Technical Report, University of Cambridge, Computer Laboratory*, no. 941, 2019, [https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-941.pdf](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-941.pdf)

\[4\] *RISC-V International*, [http://community.riscv.org](http://community.riscv.org).

\[5\] Robert N.M. Watson, et al. "It Is Time to Standardize Principles and Practices for Software Memory Safety", *Communications of the ACM*, volume 68, issue 2, pages 40-45. [https://dl.acm.org/doi/10.1145/3708553](https://dl.acm.org/doi/10.1145/3708553)

\[6\] Jeremiassen, Tor. “MPACT-Cheriot.”, March 2024, [https://github.com/google/mpact-cheriot](https://github.com/google/mpact-cheriot).

\[7\] Jeremiassen, Tor. “MPACT-RiscV.”, March 2023, [https://github.com/google/mpact-riscv](https://github.com/google/mpact-riscv).

\[8\] Jeremiassen, Tor. “MPACT-Sim \- Retargetable Instruction Set Simulator Infrastructure.”, March 2023, [https://github.com/google/mpact-sim](https://github.com/google/mpact-sim).

\[9\] "Protocol Buffers \- Google's data interchange format", [https://github.com/protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf).

\[10\] "ReNode", [https://renode.io](https://renode.io).

\[11\] "RISC-V Advanced Core Local Interruptor Specification", [https://github.com/riscv/riscv-aclint](https://github.com/riscv/riscv-aclint).

\[12\] "RISC-V Platform-Level Interrupt Controller Specification", [https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc](https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc).

\[13\] "RISC-V Semihosting", [https://github.com/riscv-non-isa/riscv-semihosting](https://github.com/riscv-non-isa/riscv-semihosting).

\[14\] “Semihosting.” *RealView Compilation Tools Developer Guide Version 4.0*, ARM Ltd, Chapter 8, [https://developer.arm.com/documentation/dui0203/j](https://developer.arm.com/documentation/dui0203/j).

\[15\] Joannou, Alexandre, et al. “Randomized Testing of RISC-V CPUs using Direct Instruction Injection.” *IEEE Design and Test*, 2023, [https://www.repository.cam.ac.uk/bitstream/handle/1810/347706/Randomized\_Testing\_of\_RISC-V\_CPUs\_using\_Direct\_Instruction\_Injection.pdf](https://www.repository.cam.ac.uk/bitstream/handle/1810/347706/Randomized_Testing_of_RISC-V_CPUs_using_Direct_Instruction_Injection.pdf).

\[16\] Gray, K. E., Sewell, P., Pulte, C., Flur, S., & Norton-Wright, R. (2017). *The Sail instruction-set semantics specification language* (pp. 1-25). Technical report, Cambridge University, 2017, [https://www.cl.cam.ac.uk/\~pes20/sail/manual.pdf](https://www.cl.cam.ac.uk/~pes20/sail/manual.pdf).

\[17\] "CHERIoT RTOS", [https://github.com/CHERIoT-Platform/cheriot-rtos](https://github.com/CHERIoT-Platform/cheriot-rtos).

[image1]: <data:image/png;base64,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>