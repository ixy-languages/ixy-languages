Overview
=========

Ixy is an educational user space network driver for the Intel ixgbe family of 10 Gbit/s NICs (82599ES aka X520, X540, X550, ...).
Its goal is to show that writing a super-fast network driver can be surprisingly simple, check out the [full description in the main repository](https://github.com/emmericp/ixy).

[Check out our talk at 35C3](https://media.ccc.de/v/35c3-9670-safe_and_secure_drivers_in_high-level_languages) (Video, 60 minutes)

Ixy was originally written in C as lowest common denominator of system programming languages, but it is possible to write user space drivers in any programming language.

Yes, these drivers are really a full implementation of an actual PCIe driver in these languages; they handle everything from setting up DMA memory to receiving and transmitting packets in a high-level language. You don't need to write any kernel code to build drivers!
Some languages require a few lines of C stubs for features not offered by the language; usually related to getting the memory address of buffers or poking MMIO registers in the right way. But all the core logic is in high-level languages; the implementations are about 1000 lines of code each.

| Language | Code                                                    | Status   | Full evaluation | 
|----------|---------------------------------------------------------|----------|-----------------|
| C        | [ixy.c](https://github.com/emmericp/ixy)*                | Finished | [Paper (draft)](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/ixy_paper_draft2.pdf) |
| Rust     | [ixy.rs](https://github.com/ixy-languages/ixy.rs)       | Finished | [Thesis](https://www.net.in.tum.de/fileadmin/bibtex/publications/theses/2018-ixy-rust.pdf) |
| go       | [ixy.go](https://github.com/ixy-languages/ixy.go)       | Finished | [Thesis](https://www.net.in.tum.de/fileadmin/bibtex/publications/theses/2018-ixy-go.pdf)
| C#       | [ixy.cs](https://github.com/ixy-languages/ixy.cs)       | Finished | [Thesis](https://www.net.in.tum.de/fileadmin/bibtex/publications/theses/2018-ixy-c-sharp.pdf)
| Swift    | [ixy.swift](https://github.com/ixy-languages/ixy.swift) | Finished      | [Documentation](https://github.com/ixy-languages/ixy.swift/blob/master/README.md)             |
| OCaml    | [ixy.ml](https://github.com/ixy-languages/ixy.ml)       | Working but unoptimized      | [Documentation](https://github.com/ixy-languages/ixy.ml/blob/master/README.md)             |
| Haskell  | [ixy.hs](https://github.com/ixy-languages/ixy.hs)       | Working but unoptimized      | WIP             |
| Python   | [ixy.py](https://github.com/ixy-languages/ixy.py)*      | 80% finished      | WIP             |

*) also features a VirtIO driver for easy testing in VMs with Vagrant


This repository here is only a short summary of the project, check out the repositories and full evaluations linked above for all the gory details.


Performance
============
Our [benchmarking script](https://github.com/ixy-languages/benchmark-scripts) for the [MoonGen packet generator](https://github.com/emmericp/MoonGen) loads the forwarder example application with full bidirectional load at 20 Gbit/s with 64 byte packets (29.76 Mpps).
The forwarder then increments one byte in the packet to ensure that the packet is loaded all the way into the L1 cache.
Correct functionality of the forwarder is also tested by the script by validating sequence numbers.

Running this on a single core of a Xeon E3-1230 v3 CPU yields these results when varying the CPU frequency.

![CPU frequency vs. throughput](img/cpufreq.png)

A main driver of performance for network drivers is sending/receiving packets in batches from/to the NIC.
Ixy can already achieve a high performance with relatively low batch sizes of 32-64 because it is a full user space driver.
Other user space packet processing frameworks like netmap that rely on a kernel driver need larger batch sizes of 512 and above to amortize the larger overhead of communicating with the driver in the kernel.

Effect of batch size with CPU clocked at 3.3 GHz
![Performance with different batch sizes, CPU at 3.3 GHz](img/batchsize-3.3.png)

Effect of batch size with CPU clocked at 1.6 GHz
![Performance with different batch sizes, CPU at 1.6 GHz](img/batchsize-1.6.png)

[TODO: measure cache misses for other languages, similar to the cache measurement for the C version in [the paper](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/ixy_paper_draft2.pdf)]

*Notes on multi-core:* Some languages implement multi-threading for ixy, but some can't or are limited by the language's design (e.g., the GIL in Python and OCaml). However, this isn't a real problem because multi-threading within one process isn't really necessary.
Network cards can split the traffic at the hardware level (via a feature called RSS), the traffic can be distributed to independent different processes.
For example, [Snabb](https://github.com/snabbco/snabb) works like this and many DPDK applications use multiple threads that do not communicate (shared nothing architecture).



Latency
=======

Average and median latency is the same regardless of the programming language as latency is dominated by buffering times, the [evaluation script](https://github.com/ixy-languages/benchmark-scripts) can sample the latency of up to 1000 packets per second with hardware timestamping (precision: 12.8 nanoseconds).
This yields somewhat interesting results depending on queue sizes and NUMA configuration, see the [ixy paper](https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/ixy_paper_draft2.pdf) for an evaluation.

Latency spikes induced by languages feature a garbage collector and/or a JIT compiler might not be caught by the test setup above.
We have a second test setup to catch this: we also capture *all* packets before and after the device under test with fiber taps and use the  [MoonSniff framework](https://github.com/AP-Frank/MoonGen/tree/moonsniff) to acquire all hardware timestamps for all packets.

Running this for Rust, Go, and C# at 18 Mpps yields the following latency distribution:

![Latency](img/latency.png)

This graph focuses on the tail latency, note the logarithmic scaling on the x axis.
