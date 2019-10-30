# Why is Rust slightly slower than C?

Context: Check out the [README file](README.md) first. This is an adapted excerpt from our research paper ["The Case for Writing Network Drivers in High-Level Languages"](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/the-case-for-writing-network-drivers-in-high-level-languages.pdf) [[BibTeX](https://www.net.in.tum.de/publications/bibtex/highleveldrivers.bib)].

Our Rust driver is a few percent slower than our C driver, why? Well, it's of course because of safety features in Rust, but can we quantify that?

There are only two major differences between the idiomatic Rust and C implementations:

1. Rust enforces bounds checks while the C code contains no bounds checks (arguably idiomatic style for C).
2. C does not require a wrapper object for DMA buffers, it stores all necessary metadata directly in front of the packet data the same memory area as the DMA buffer.

However, the Rust wrapper objects can be stack-allocated and effectively replace the pointer used by C with a smart pointer, mitigating the locality penalty.
The main performance disadvantage is therefore bounds checking.

## CPU performance counters

Looking at CPU performance counters while forwarding packets shows interesting results.
This table shows relevant performance counters divided by forwarding speed, i.e., in events per packet.


| **Counter** \ **Lang**| C, batch 32 | Rust, batch 32 |    | C, batch 8 | Rust, batch 8 |
|-----------------------|-------------|----------------|----|------------|---------------|
| Cycles                | 94          | 100            |    | 108        | 120           |
| Instructions          | 127         | 209            |    | 139        | 232           |
| IPC                   | 1.35        | 2.09           |    | 1.29       | 1.93          |
| Branches              | 18          | 24             |    | 19         | 27            |
| Branch mispredicts    | 0.05        | 0.08           |    | 0.02       | 0.06          |
| Store µops            | 21.8        | 37.4           |    | 24.4       | 43.0          |
| Load µops             | 30.1        | 77.0           |    | 34.4       | 84.2          |
| Load L1 hits          | 24.3        | 75.9           |    | 28.8       | 83.1          |
| Load L2 hits          | 1.1         | 0.05           |    | 1.2        | 0.1           |
| Load L3 hits          | 0.9         | 0.0            |    | 0.5        | 0.0           |
| Load L3 misses        | 0.3         | 0.1            |    | 0.3        | 0.3           |

Wow, looks like we should really be asking how Rust manages to be so fast! The CPU executes 65% more instructions per forwarded packet with the Rust driver compared to C.
But it manages to do so with only 6% (11%) more cycles per packet for batch size 32 (batch size 8).

Modern superscalar out-of-order CPUs (this was run on an Intel Xeon E3-1230 v3) are really good at hiding these safety checks. The C code just didn't use the CPU to it's full potential.
Our driver does not violate any bounds checks during normal execution, the processor is therefore able to correctly predict (branch mispredict rate is at 0.2% - 0.3%) and speculatively execute the correct path.
Caches also help with the additional required loads of bounds information: this workload achieves an L1 cache hit rate of > 98%.

## Integer overflow checks

Another safety feature in Rust are integer overflow checks that have to be explicitly enabled with a compile-time flag.
Doing so decreases throughput by only 0.8% at batch size 8, no statistically significant deviation was measurable with larger batch sizes.

Profiling shows that 9 additional instructions per packet are executed by the CPU with overflow checks, 8 of them are branches.
Total branch mispredictions are unaffected, i.e., the  branch check is always predicted correctly by the CPU.

## Rust without safety checks

We can use [C2Rrust](https://github.com/immunant/c2rust) to transpile our C code to Rust, yielding a completely unsafe Rust implementation that features barely any safety checks. The resulting transpiled code forwards a packet in only 91 cycles while executing 121 instructions, that's faster than the original code. It holds true even when the original code is compiled with clang and the same LLVM version.

Well, there you have it: **Rust is faster than C!** (unless you want safety)


