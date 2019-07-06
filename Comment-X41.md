Comments from @tevador.

## 4.1. Findings

The 4 listed vulnerabilities are real issues that must be fixed. However, I think the report should explicitly state that these vulnerabilities are only manifested when using a non-standard set of compile-time parameters (the Monero parameters are not affected).

Additionally, vulnerabilities RNDX-PT-19-01, RNDX-PT-19-02 and RNDX-PT-19-04 are all caused by using an excessive program size. The crash in the 4th issue is caused by an overflow in the branch target bytecode field, which is only a 16-bit number. Therefore, we will explicitly limit the program size to a maximum of 32768 instructions with a static_assert to prevent this scenario.

## 4.2. Feasibility of Implementing RandomX in Hardware

RandomX branches are not taken with a probability of 255/256 ~ 0.9961. It should be noted that branch predictors are actually less efficient than no prediction if the branch condition is completely random like in RandomX. See Agner Fog: The microarchitecture of Intel, AMD and VIA CPUs - https://www.agner.org/optimize/microarchitecture.pdf on page 15
For example, if the not taken probability is 90%, the experimental misprediction rate of a CPU predictor is 11%, which is worse than if all branches were predicted "not taken" every time (then the misprediction rate would be 10%).
We are mitigating this effect by using a very low branching probability.

So while it's true than the branch predictor unit is a rather large part of the CPU core, we have not found a way how to meaningfully utilize it (we have tried).

We have already commented about the fact that reordering across branch targets is generally not possible.

Increasing data dependency between instructions would also hurt CPU performance because modern superscalar CPUs usually have a 4-wide execution engine, so the current parallelism fits this.
Increasing the number of programs per hash would increase the overhead of JIT compilation, so it's not a silver bullet either.
Self-modifying programs are not feasible because we need a compilation step.
There is no such thing as a multicore program. Every program can be executed serially. In fact, PoW is already the best parallelizable workload because each thread works on different hash.
Privileged instructions, interrupts, virtual memory and hardware virtualization are not useful in a PoW algorithm. These tools help CPUs to multitask, but PoW only cares about the final hash value, which depends on the sequence of math operations and not on the underlying protocols.

## 4.3. Weaknesses in the Cryptographic Implementations and Algorithms

AesHash1R is not a cryptographic hash, but serves as a universal hash. We are aware that it's easily invertible. Same with the AES generators.

RNDX-PT-19-105: Insufficient Diffusion in AesGenerator4R - this was already identified by another audit team and is fixed as of RandomX v1.0.4.

RNDX-PT-19-106: Poor Code Coverage - RandomX v1.0.4 contains a test suite that covers most basic steps of the algorithm.

RNDX-PT-19-107: JIT Memory Pages for Generated Code are Writable and Executable - we don't think this is exploitable and it's done mainly for performance reasons.

RNDX-PT-19-108: Sandboxing RandomX Execution - The way RandomX programs are constructed, it is not possible to execute arbitrary code or act on arbitrary data.

RNDX-PT-19-109: Incorrect SuperScalarHash Latency when Program Size is too Small - this has been fixed some time ago. RANDOMX_SUPERSCALAR_MAX_SIZE is now calculated based on RANDOMX_SUPERSCALAR_LATENCY.

