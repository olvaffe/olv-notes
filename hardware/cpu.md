CPU
===

## Parallel Computer Architecture and Programming, CMU

- Lecture 1
  - History of Processor Performance
    - data paths: 4-bit -> 8-bit -> 16-bit -> 32-bit -> 64-bit
    - efficient pipelining: 3.5 CPI -> 1.1 CPI
    - instruction-level parallelism (ILP): superscalar with 4
      instructions/cycle, out-of-order execution
    - clock rates: 10MHz -> 200MHz -> 3GHz
    - From 2004 and on, because of thermal wall,
      - clock rate stops increasing
      - power usage stops increasing
      - no further benefit from ILP
      - this is per core!
    - switches to multi-core and parallel programming
  - Parallel computer
    - 2015 Intel Skylake
      - 4 cores
    - 2017 nVidia Volta
      - 80 cores
    - keys to performance
      - minimize the cost of inter-core communications
      - improive distribution of works between cores
- Lecture 2
  - CPU: dummy SW, smart HW
    - write a serial program and CPU dynamically schedules instructions
    - small areas of the core are for execution
    - most areas of the core are scheduling
    - rest are for caches, I/O
  - GPU: smart SW, dummy HW
    - write a parallel program and GPU executes it efficiently
    - most areas of the core are for execution
    - small areas of the core are for caches, I/O
    - very little for scheduling
  - Given a simple four-stage processor that executes in program order
    - fetch -> decode -> execute -> commit
    - each stage takes one cycle
    - latency: 4cycles, throughput: 0.25
  - Pipelining
    - latency: 4cycles but throughput is 1!
    - N-stage pipeline gives N times throughput
    - requires consecutive instructions to be independent
    - data hazards
      - c=a+b
      - d=a+c
      - the second add needs to wait for the first add
      - data forwarding reduces the stall
        - after the first add is executed, move on to the commit stage but
          also feed the result to the decode stage for the following add
    - branch prediction
      - when a branching instruction hits the execute stage and results in a
      	jump, the processor needs to throw away the instructions in the fetch
      	and decode stages
      - for a deeply pipelined processor, they are wasted works
      - modern processor speculates (i.e., guesses) where to go next before
      	fetching with >95% prediction accuracy
  - out-of-order execution
  - superscalar
  - it is so hard that it does not scale
  - moving to multi-core
- Lecture 3
  - SIMD: have multiple EUs execute the same instruction to achieve SIMD
  - SMT
  - Memory Latency
    - caches
    - prefetching
    - latency hiding: run multiple instruction streams in a single core
      - In GPU world, a core has a limited number of registers (sRAM).  The
      	number of registers for each stream is known.  It can determine how
      	many streams are allowed in this single core.
  - memory bandwidth limited
    - when a program is optimized for throughput, the memory bandwidth is
      usually the bottleneck

## Scalar

- most basic design, with one CU (control unit) and one EU (execution unit)
- CU fetches and decodes instructions one at a time
- EU executes instructions one at a time
- latency: how many cycles to execute an instruction
- throughput: how many instructions can be executed per cycle
- on a basic scalar processor, throughput == 1/latency

## Pipelining

- divide EU into many (~30) stages
- when instruction I is being executed in stage S, instruction I+1 can be
  executed in stage S-1
- latency is the same as the basic scalar processor
- but when fully pipelined, throughput is much higher (~x30) than a basic
  scalar processor

## Superscalar

- have multiple (~4) EUs
- maybe multiple (~2) CUs, but fetching from the same instruction stream
- when there is no dependency between consecutive insructions, they can be
  executed concurrently
- latency is the same as the basic scalar processor
- but with full concurrency, throughput is multipled by the number of EUs

## SIMD

- EU is extended such that an instruction can operate on a small, fixed-size
  (16, with AVX-512) vector of data
- sometimes, it just means that multiple EUs are used to execute the same
  instruction to achieve SIMD

## SMT (Simultaneous multithreading)

- have multiple (~2) CUs fetching from different instruction streams
- depend on superscalar; thus CUs can share the EUs

## TMT (Temporal multithreading, or super-threading)

- have rapid thread context switch
- interleave instructions from mulitple threads

## Register Files and Caches

- register files and caches are implemented with sRAM
  - register/L1d/L1i/L2$ are per-core
  - L3$ is shared by all cores
- register files are optimized for speed (at the cost of power/heat/die-space)
- caches are optimized for die-space and then power (at the cost of speed)
- registers and L1 are also closer to EUs than L2/L3

## system-on-chip

- traditionally, there are two core-logic chipsets outside of the CPU
  - northbridge is connected to the CPU using FSB and serves as the controller
    for RAM, AGP, high-speed PCI-e
  - southbridge is connected to northbridge and serves as the controller for
    low-speed buses
- overtime, things moved into a SoC design
  - there are several CPU cores, each with its own frontend, execution engine,
    and memory subsystem (L$/L2)
    - the cores are connected together with a ring interconnect
  - there is a separate huge GPU core
  - both CPU cores and GPU core are connected to a shared L3
  - there is a system agent, evolved from northbridge, that sits between the
    L3 and the I/O buses
  - southbridge evolved into platform controller hub, PCH, and is slowly
    moving into the system agent

## Caches

- Four-way set associative cache
  - each memory block can be cached in four possible row entries of the cache
  - each cache row entry consists of
    - tag
    - data block, usually 64 bytes
    - flag bits
  - e.g., a four-way set associative cache of size 8KB and 64-byte cache
    blocks has
    - 8KB/64 = 128 cache blocks
    - 128/4 = 32 sets
    - a 32-bit memory address is splitted into 21bit tag, 5bit index, and 6bit
      offset
      - every 2KB of memory share the same tag
      - every 64B goes to a different set
- VIPT, virtually indexed and physicalled tagged
  - Intel
  - ARM I$
- PIPT, physically indexed and physically tagged
  - ARM D$

## Power Dissipation

- <https://en.wikipedia.org/wiki/Processor_power_dissipation>
  - dynamic power consumption is proportional to `V^2 * freq`
  - assuming `V` is proportional(!?) to `freq`, dynamic power consumption is
    proportional to `freq^3`
  - since runtime is inversely proportional to `freq`, the total power
    consumption is proportional to `freq^2`
- e.g.,
  - `Ryzen 7 7700X` runs at 4.5GHz and TDP is 105W
  - `Ryzen 7 7700` runs at 3.8GHz and TDP is 65W
- e.g.,
  - `i5-13400` runs at 2.5/1.8GHz and TDP is 65W
  - `i5-13400T` runs at 1.3/1.0GHz and TDP is 35W

## Die Cost

- for 3nm process,
  - the cost is about $20K
  - the yield rate is about 80%
  - a wafer is typically 300mm (diameter)
    - also known as 12"
- die size
  - it varies, but the ballpark number is 110~170mm2
  - it is easy to calculate the die cost from process cost, wafer size, yield
    rate, and die size
  - an soc die typically has these ip blocks
    - cpu cores
    - per-core l1/l2 caches
    - shared l3 caches
    - gpu
      - shader engines
      - media engine
      - display engine
    - npu
    - io
      - pcie
      - usb
      - misc
    - wifi, bt, modem, etc.
- standard cells
  - they are like primitives to build logic gate, io pad, sram, clock, etc.
  - they can be optimized for performance or die area
    - HD: high-density, optmized for perf-per-mm2
    - HP: high-performance, can reach much higher performance but with much
      bigger area
