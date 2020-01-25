---
title: "Graphcore: headerless packets and the perfect NOC"
date:  2020-01-25 20:32:40 -00:00
categories: Networks Graphcore
layout: post
---

# Introduction

Network on chip (NoC) designs have been relatively isolated from the software that runs using them, treated as an implementation detail while the instruction set and memory model are exposed to the programmer. Graphcore has totally exposed the NoC to the program, and in doing so routes packets without headers.

# Graphcore?

Graphcore is a Bristol, UK based chip design startup developing a ML accelerator equalling current NVIDIA parts for direct matrix multiply performance but (probably) maintaining that same performance on less structured workloads. This is in contrast to the standard GPGPU model that suffers significant performance degradation on programs with unpredictable control flow.

Chip multiprocessors need to ensure the interconnect can sustain high instruction throughput for the cores at the minimum area and power cost for a target workload. For a CPU design, this will often take the form of an interconnected series of tradeoffs between cache size and behaviour, interconnect design around coherency, bisection bandwidth, topology, and routing strategy, and the downstream memory controllers. This is made significantly more difficult by the target market for a CPU requiring improved performance often for the same binaries, so chip designers are constrained not only by technological limitations but also by the optimisation decisions of the past.

Prior general purpose chip designs have targeted high single-thread performance at the expense of parallelism or bisection bandwidth. This has lead to common designs for network-on-chip systems targeting low area - from busses for very low core counts, to ring designs for ~10 core ships (IBM CELL, Xeon 8-core), then (modified) mesh layouts with packet routing for larger designs.

GPGPU designs modify this by adding many more processing elements than the memory controllers could support for CPU code, and relying on the programmer to allocate tasks that are able to saturate the vast number of ALUs without requiring communication or irregular memory access to fetch new data or code. The typical example is a matrix multiply, where for the naive implementation the kernel size is constant, caches can be fully utilised, and memory access is totally regular.

# The Graphcore and Celerity network designs

The Graphcore targets a different design point - moderately branching, highly parallel, ultra-highly bandwidth intensive algorithms. A possible example is page rank on a dense graph. This workload requires updates to be sent along every edge for each iteration for the classic implementation. This allows a very small amount of data to overwhelm nearly any interconnect.

Another chip design targeting the same tightly integrated mesh of weak cores is the Celerity manycore RISC-V accelerator chip. Celerity is a tiered design incorporating 5 Rocket and 496 Vanilla-5 cores + a specialisation layer. The 496 Vanilla-5 cores form a mesh that provides a useful design point comparison with the Graphcore, but performance comparisons would be misleading as the Grahcore uses a entire ~800mm^2 TSMC 16nm reticle - whereas the Celerity mesh is tapped out into 15.24 mm² of silicon.

I believe both networks use a 2D grid of connections between routers, and a single core per router design. Where Graphcore and Celerity differ is the programming model exposed by this mesh. 

Celerity packets are single flits, including a 32bit message header and 32bit payload. There is no wormhole routing or virtual circuits - each packet/flit is individually routed. This enables transfers to instantly hit full bandwidth, but conversely halves the peak bandwidth for a given number of wires vs a virtual circuit design. This is an aggressive but not revolutionary design point, suitable for the very tight delivery timelines within the DARPA Circuit Realization At Faster Timescales (CRAFT) project.

Graphcore also uses a mesh NOC, but emulates an all-to-all crossbar at O(N) wire cost with compiler assistance. A graph core packet is also a single flit, but has no header at all. This makes it impossible to route a packet without external help. Instead, the cores are devoted to implementing a totally predetermined routing pattern during communication portions of the program.

The Graphcore programming model flows a Bulk Synchronous Parallel model: on N cores, N threads make progress up to a communications point and then block. Once all threads have reached the sync point the entire processor transitions into a communications period simultaneously.

During the communications period, cores operate in lockstep. Each clock period, a core may send a flit, store a flit from one port into a register slot, reconfigure it's router's forwarding table, or all of the above. 
 
The on-chip interconnect is time-deterministic and uncontended. Using this property, the graph core compiler has produced code for the communication phase that emits and receives messages over the interconnect according to a core-local schedule. There is no method for to be notified when a core receives a flit - the program makes use of a global clock within the communication period to count cycles, and will directly copy the data on the input bus into local cache when a valid message is scheduled to be received.

The transmission latency depends on the source and destination tile id. A recent benchmark paper demonstrated very regular patterns to the latency that probably enables effective compiler heuristics for arranging communicating threads.

In order to correctly route packets without a header each intermediary core will reconfigure its router to correctly forward incoming data. The compiler has produced a complete schedule for this phase and so is able to emit this instruction sequence. This is how the graph core interconnect can appear to be a full crossbar at O(N^2) wire cost, whilst only using O(N) wire resources - compiler scheduling will avoid packet conflicts by using many possible routes to each core. Each core can only send unicast messages so the performance difference is not observable.

Graphcore puts emphasis on the BSP model not sacrificing performance as it may be impossible to power both the cores and interconnect simultaneously. This sounds implausible as the cores occupy significantly more area and operate on the same clock. A more accurate statement may be that using the communication phase, the ALUs are powered down and routers powered up - with the cores still performing instruction decode and register operations for the communications sequence.

The Graphcore design is a significant innovation in hardware/compiler codesign and will offer a unique performance tradeoff, with programmers able to use huge bandwidth and ultra low latency interconnects for massively parallel programs - a offering that was previously assumed to be impossible!

### References

https://fuse.wikichip.org/news/3217/a-look-at-celeritys-second-gen-496-core-risc-v-mesh-noc/

Synchronisation in a Multi-Tile processing arrangement, GB2569269, S. Knowles, A. Alexander, 2017

