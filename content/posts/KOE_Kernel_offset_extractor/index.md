---
title: "KOE: Kernel Offset Extractor"
date: 2026-04-06
draft: false
tags: ["Linux Kernel", "Reverse Engineering", "Firmware"]
summary: "How I locate functions inside stripped Linux kernels by comparing them with a recompiled reference kernel"
---

## Introduction

While developing CFuzz, I wanted to emulate the original Linux kernels extracted from vendor firmware. Keeping the vendor kernel is useful because it preserves the behavior of the real device, including its patches, drivers, and quirks. Unfortunately, firmware images rarely come with a symbol table or debugging information. They may also be packed, encrypted, or stored in a vendor-specific format.

This becomes a problem as soon as the emulator needs to interact with the kernel. CFuzz, for example, hooks functions such as `read`, `recv`, and `connect` to detect when a process is ready to consume an input. QEMU needs an address at which to place that hook, but a stripped kernel no longer tells us where those functions are.

If I can recompile the kernel, the solution is straightforward: enable debug information and obtain the symbol addresses with tools such as `readelf` or `objdump`. The interesting case is when the original vendor kernel boots correctly in QEMU and I do not want, or cannot, replace it.

That is the problem I built KOE, the Kernel Offset Extractor, to solve.

## The main idea

Stripping a binary removes its names and debugging metadata, but it does not remove its machine code. The compiled implementation of a function is still there; we just do not know which block of instructions belongs to which source-level name.

KOE uses a second kernel as a reference. I compile a kernel with a version and configuration as close as possible to the vendor one, this time keeping its symbols. I can then extract the assembly of the function I am looking for from the reference build and compare it with every function-like block found in the stripped image.

The complete pipeline looks like this:

```text
Stripped vendor kernel                      Recompiled reference kernel
        |                                               |
        v                                               v
Disassemble the entire image                Extract the named function
        |                                               |
        v                                               v
Split it into candidate blocks              Build its instruction fingerprint
        |                                               |
        +--------------------+--------------------------+
                             |
                             v
                  Compare all fingerprints
                             |
                             v
             Return the lowest-scoring candidate
```

This is not symbol recovery in the traditional sense. KOE answers a narrower question: *which block in this stripped kernel most closely resembles this known function?*

## Finding function boundaries

The first step is to turn one large disassembly into a list of candidate functions. Compilers usually emit recognizable function prologues and epilogues. Reverse-engineering tools use several heuristics of this kind to recover function boundaries when symbols are unavailable.

The exact pattern is architecture-dependent. On MIPS, which is the example used here, a common function prologue allocates a stack frame with an instruction such as:

```asm
addiu sp, sp, -32
```

More generally, the pattern is `addiu sp, sp, -N`, where `N` is the size of the stack frame. KOE disassembles the raw kernel with the appropriate cross-architecture `objdump`, finds occurrences of that pattern, and starts a new block at each occurrence. In simplified Python, the splitting logic is:

```python
def split_chunks(assembly, prologue_pattern):
    chunks = []
    current_chunk = []

    for line in assembly.strip().splitlines():
        if prologue_pattern.search(line) and current_chunk:
            chunks.append("\n".join(current_chunk))
            current_chunk = []

        current_chunk.append(line)

    if current_chunk:
        chunks.append("\n".join(current_chunk))

    return chunks
```

For example, one recovered block may look like this:

```asm
8020f664:  27bdffe0  addiu  sp,sp,-32
8020f668:  afbf001c  sw     ra,28(sp)
8020f66c:  0c083d46  jal    8020f518
8020f670:  afa50010  sw     a1,16(sp)
8020f674:  8fa50010  lw     a1,16(sp)
8020f678:  aca20180  sw     v0,384(a1)
8020f67c:  8fbf001c  lw     ra,28(sp)
8020f680:  03e00008  jr     ra
8020f684:  27bd0020  addiu  sp,sp,32
```

This heuristic is deliberately simple. It works well for many conventional MIPS functions, but it is not a complete disassembler: leaf functions may not allocate a stack frame, optimized functions may use a different prologue, and data embedded in executable regions can confuse the split. These limitations become important when interpreting the final score.

## Building a function fingerprint

Comparing raw bytes would be too strict. A different load address changes branch targets, configuration options change constants, and even a small source modification can shift registers or stack offsets. Exact byte equality would reject two functions that are structurally very similar.

Instead, KOE discards operands and counts instruction mnemonics. The block above becomes:

```text
{
  addiu: 2,
  sw:    3,
  jal:   1,
  lw:    2,
  jr:    1
}
```

The same transformation is applied to every candidate block in the vendor kernel and to the target function extracted from the reference `vmlinux`. Because the reference kernel contains symbols, its block can be obtained directly with a command such as:

```bash
mips-linux-gnu-objdump --disassemble=FUNCTION_NAME vmlinux
```

The result is a compact fingerprint: a histogram describing which kinds of instructions the function contains and how frequently each one appears.

Ignoring operands makes the comparison tolerant of changed addresses and immediate values. Ignoring instruction order makes it more tolerant still, but also less precise. Two functions with very different behavior can have the same instruction histogram, especially when they are short. This is the central trade-off of the heuristic.

## Scoring the candidates

Let `T` be the fingerprint of the reference function and `C` the fingerprint of a candidate. KOE computes the absolute difference for every mnemonic present in either block and sums the results:

```text
score(T, C) = sum(abs(T[opcode] - C[opcode]))
```

A perfect match has a score of zero. Lower values indicate a closer match.

Suppose the candidate and reference fingerprints are:

```text
candidate = {addiu: 2, sw: 3, jal: 1, lw: 2, jr: 1}
reference = {addiu: 2, sw: 3, jal: 1,        jr: 3, move: 1}
```

Their difference is:

```text
{addiu: 0, sw: 0, jal: 0, lw: 2, jr: 2, move: 1}
```

The final score is therefore `5`. KOE repeats this calculation for the thousands of blocks found in the stripped kernel, ranks them by score, and selects the closest candidate. The address of the first instruction in that block is the candidate function address.

The algorithm is intentionally inexpensive. Once the kernel has been disassembled and fingerprinted, searching for another function only requires extracting one reference block and comparing small dictionaries.

## Why the reference build matters

The quality of the match depends heavily on the reference kernel. Ideally, it should use:

- the same Linux version;
- the same architecture and endianness;
- a similar kernel configuration;
- the same compiler family and version;
- equivalent optimization options.

In practice, vendor build environments are often unavailable. A kernel built from the corresponding source version with a standard or recovered configuration can still provide a useful template, but the score becomes less trustworthy as the two builds diverge. Configuration changes may enable different code paths, compiler versions may select different instructions, and vendor patches may alter the target function itself.

One way to reduce this dependency is to generate several reference builds with different compiler versions or optimization settings. If the same candidate consistently ranks first across multiple templates, that is much stronger evidence than a single low score.

## From an extracted address to a runtime hook

There are three values that are easy to confuse: the byte offset inside a file, the virtual address shown by the disassembler, and the address used by the running kernel. They are not always identical.

Before using a KOE result, I need to account for the format and loading process of the kernel:

- `uImage` includes a container header;
- `zImage` contains a compressed, self-extracting kernel;
- `vmlinux` is normally an ELF file with section virtual addresses;
- raw images may require an explicit disassembly base address;
- relocation or KASLR may add a runtime load bias.

KOE locates the function relative to the image it analyzed. If QEMU loads that image at a different base, the candidate must be translated to its runtime address. For many older embedded kernels this mapping is stable, but it should never be assumed without checking.

Once the address is resolved, CFuzz can use it as a hook point. In my use case, the interesting targets are system calls involved in input handling. Reaching `recv`, for example, tells the framework that a process is about to receive external data. At that point CFuzz can take a snapshot, repeatedly restore it, inject mutated inputs, and collect process-specific coverage without rebooting the whole firmware for every test case.

This is why recovering one function address is valuable: KOE provides the bridge between a stripped vendor kernel and the runtime instrumentation needed by the fuzzer.

## Validating a match

The best score is a candidate, not proof. I validate an extracted offset before relying on it:

1. Inspect the top candidates rather than looking only at the first one. A large gap between the best and second-best scores is more convincing than several ties.
2. Compare the surrounding instructions and calls. Even when operands are ignored during scoring, they contain useful context during manual validation.
3. Boot the kernel in QEMU and trace the candidate while triggering the expected behavior in the guest.
4. For a system call, verify that the hook is reached by the expected process and that its arguments look plausible.
5. When possible, repeat the search using more than one reference build.

Dynamic validation is slower than static matching, but it only needs to be performed for a small set of highly ranked candidates. This combination keeps the search fast without blindly trusting a heuristic.

## Limitations and possible improvements

KOE works because compiled versions of the same function usually retain enough structure to be recognizable. There are, however, several cases in which the current fingerprint can fail:

- very small wrappers produce many collisions;
- inlined functions do not exist as independent blocks;
- leaf functions may not match the expected prologue;
- compiler and configuration differences can substantially change the instruction mix;
- the wrong architecture, endianness, or image base makes the disassembly meaningless;
- syscall names and wrapper layers change between Linux versions;
- a histogram cannot distinguish two sequences containing the same instructions in a different order.

The natural next step is to preserve more structure without returning to fragile byte matching. Ordered opcode n-grams could capture local instruction order, while a lightweight control-flow fingerprint could represent branches and calls. Rare instructions could also receive more weight than common instructions such as loads and stores. Finally, reporting a top-k list and a confidence margin would make ambiguous results explicit.

## Conclusion

KOE started from a practical problem: I had a stripped kernel that already worked in QEMU, but CFuzz needed function addresses to instrument it. Replacing the kernel would have changed the environment I wanted to test, while manually reversing every firmware would not scale.

The extractor offers a useful middle ground. It recovers candidate function locations by combining architecture-specific boundary detection with a tolerant instruction fingerprint and a recompiled reference kernel. It is not a replacement for a full reverse-engineering suite, and its output still needs validation, but it automates the repetitive part of the search and makes stripped vendor kernels much easier to instrument.
