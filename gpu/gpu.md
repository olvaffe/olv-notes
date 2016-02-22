GPU
===

## MC (Memory Controller)

* GPU has its own address space
* It has access to its VRAM and system memory through GART
* Both needs to be mapped into GPU's address space


## CPU v.s. GPU

* CPU devides itself to cache, flow control, and the ALUs
* GPU denotes most of itself to ALUs

## CUDA

* `MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C)` adds matrices `A` and `B`,
  and stores the result in `C`
  * Usually, there are 16x16 threads per block
  * Say the size of a matrix is NxN, there will be `(N/16 * N/16)` blocks
  * each block must be run by a single core
  * different blocks may be run by different cores
* thread local memory is faster than shared memory is faster than the global
  memory
* CUDA hardware is built around an array of multithreaded SMs (streaming
  multiprocessors)
* When a SM is given one or more thread blocks to execute, it partitions them
  into warps that get scheduled by a warp scheduler
  * a warp of an SM has 32 threads
