---
layout: post
title:  "FPGA Quantum Simulation"
date:   2019-01-14 08:06:15 -07:00
categories: Haskell FPGA
---
# A high performance quantum circuit simulator in CλaSH

## Introduction

For my master's project, I aimed to develop a FPGA based quantum simulator with the objective of running >30 qubits at reasonable depth on a single Amazon F1 instance. Whilst the full-scale tests never happened, a tested general simulator was produced and was validated at 16 qubits on Zynq hardware. This series of blog posts will describe the simulation method and the use of CλaSH as a tool for rapid development of hardware networks.

Quantum simulation would be better termed quantum _emulation_, as hopefully real QC's will come along and we will want to call the process of simulating physical systems on QC simulation. But with with few exceptions simulation has stuck and I hope this will not be too confusing for readers from 2025.

## Contents

1. Quantum circuits (skip this if you have QC background)
2. The recursive simulation algorithm
3. Hardware layout
4. Benchmarks

## The simulation method

A quantum state on N qubits contains information about the relative magnitude and phase of all 2^N possible bitstrings. In the general, and common, case all 2^N values need to be stored as complex numbers. For even a 50 qubit simulation this requires a prohibitively large amount of memory (on the order of petabytes, and growing exponentially). Google has proposed a 72 qubit chip, and if it is able to run a square circuit this is likely to demonstrate true quantum advantage.

In order to push the boundaries of simulation groups from Google, Alibaba and Oxford use tensor network methods to simulate up to 52 qubit systems on datacenter scale computers.

We use a method of simulation inspired by the path integral formalism of QM. The likelihood of sampling a given final state is the sum of the probabilities of all paths that could possibly result in this state. This contrasts with the time evolution view of QM, where a inital state is evolved in some environment and then the final sample probabilities are found from the final state.

For large physical systems performing the integral over all possible priors can be challenging. For quantum circuits however it is straightforward - at each layer in the circuit there is a gate of low arity. The only states that can contribute are the states that when acted upon by that gate, give the target state. 

This leads to a succinct `backwardsEvaluate` function for finding the amplitude of a given basis vector after applying a circuit:

{% highlight haskell %}
<!--backwardsEvaluate circuit inital_state target_state-->
backwardsEvaluate [] i t = 
    | i == t    = 1.0
    | otherwise = 0.0
backwardsEvaluate (gate:xs) i t =
    let
        prior_states = (possiblePriors gate t)
    in let
        prior_amplitudes = map (backwardsEvaluate xs i) prior_states
    in
        -- the final amplitude is the sum of prior amplitudes
        -- multiplied by the action of the gate on the prior state
        sum (map (*) (zip prior_amplitudes (map gate prior_states)))
{% endhighlight %}

This function works in tandem with a forwards evaluator to get the action of a circuit on a inital state.

{% highlight python %}
def simulate(circuit, state=0): # zero is the |00...00> state.
    for gate in circuit:
        sucessors = act(gate, circuit)
        amplitudes = map(backwardsEvaluate, sucessors)
        state = random.choices(sucessors,
                               weights=amplitudes.conj()*amplitudes, 
                               k=1)
    return state, amplitudes[sucessors.index(state)]
{% endhighlight %}

With a bit of work you can show that this function will return samples of the final state vector in proportion to the corresponding amplitude. This method also gives the true amplitude of that state so to sample the full vector you can call this function repeatedly until the absolute sum of amplitudes nears unity. Avoiding previously sampled elements of the state is a exercise for the reader!

The forwards function is presented in Python in order to emphasise that this function is not performance critical. In fact, in our implementation this function runs on the Zynq ARM core with only `backwardsEvaluate` built in hardware.

### Performance and tweaking

This method, if the `backwardsEvaluate` function runs in a depth-first manner, will use space linear in the depth of the circuit. The tradeoff is that circuit runtime becomes both exponential in width and depth. The runtime can be reduced to that of the naïve matrix multiplication method by memoizing `backwardsEvaluate` - and if the circuit contains separable states they will never appear in the cache resulting in memory use potentially lower than naïve methods.

The advantage this method poses for FPGA instantiation is that by distributing the recursive calls to `backwardsEvaluate` in the form of a tree caches can be inserted in ways that provide high physical locality of reference. This allows for very wide memory parallelism and a computational structure that can be mapped to any fabric layout or BRAM availability.

As compared to a direct matrix method this avoids a bottleneck on DRAM access, a key limiting factor for high performance FPGA designs. 

## The hardware

The FPGA modules are written in CλaSH, with a ad-hoc wiring generator in Python and a Verilator test suite. Functionality was also verified on a Xilinx Zynq chip at nearly 100MHz.

CλaSH generates synthesizable verilog from Haskell functions of type `State -> Input -> (State, Output)` where state is the full internal state of your module clock-to-clock. The type of the top-level function that the CPU calls is:

{% highlight haskell %}
findamp_mealy_N :: KnownNat n => ModuleState n -> Input -> (ModuleState n, Output)
{% endhighlight %}

We use KnownNat to add compile-time parameters to the module. In this case, the modules maintain a stack of evaluations to complete and `n` is the size of this stack.

## Benchmarks

The hardware design was validated on a Zynq development board running Linux. The target device was the xc7z020clg400 at a -1 speed grade. Due to area limitations, and in order to generate a fully entangled intermediate state, a circuit of width 4 and depth 12 was used. The full circuit executed in 3µs, with timing correctly predicted by the RTL model.

### Scaling estimation

In order to scale the design to multiple FPGA blades, we need to consider the total bandwidth that may be consumed communicating between parts of the design. 

Amazon offers the F1 instance type, making up to 8 FPGA blades consisting of Xilinx Ultrascale+ ZU9P FPGAs with 2586k LUTs. The blades are interconnected via a 400 Gbps bidirectional ring interconnect[@Amazon_EC2_F1_Instances]. In the worst case, a single blade may consist of many low-depth `FindAmp` modules, with minimum sized stack buffers.

If the modules are responsible for evaluations of depth 2, they will be able to process a new request every 11 cycles. For this design point, where the entire fabric is consuming bandwidth, the bandwidth requirement exceeds the available bandwidth by 70%. More reasonable layouts should therefore not be bandwidth limited on the Amazon FPGA service.