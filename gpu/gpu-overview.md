GPU
===

## MC (Memory Controller)

* GPU has its own address space
* It has access to its VRAM and system memory through GART
* Both needs to be mapped into GPU's address space

## CPU vs. GPU

* CPU devides itself to cache, flow control, and the ALUs
* GPU devotes most of itself to ALUs
* start with a CPU scalar core
  * one CU and one EU
  * if we can get rid of cache, out-of-order, branch predictor, memory
    pre-fetcher, the core becomes very simple, efficient, and affordable
* let's have 16 cores
  * each CU in the cores share the same instruction stream
  * if we can put more EUs into each core, we can share the CU
* let's have 8 EUs in each core
  * in each core, it is SIMT (single instruction, multiple threads)
  * wasted EUs if only some of them are active because of branching
  * stalled CU/EUs if some of them execute high latency operation (e.g.,
    texturing)
* let's context switch (super-threading) between 4 instruction streams in each
  core
  * it can execute `16*8*4` = 512 threads concurrently.
  * some are interleaved
* Fermi (2010)
  * a core has 2 CUs and 32 EUs; it can interleave 48 instruction streams
  * there are 15 cores: `15*32*48` = 23040 threads!
* Turing (2018)
  * each core has 4 CUs and 64 EUs; it can interleave 128 instruction streams
  * there are 68 cores: `68*64*128` = 557056 threads

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

## Binding Models

 - Fixed-Function
   - `glUniform1f(loc, val)` becomes `writeReg(REG_UNIFORM_<loc>,  val)`
   - `layout(location = loc) uniform float v` in GLSL becomes
     `float v = readReg(REG_UNIFORM_<loc>)`
   - `glActiveTexture(GL_TEXTURE0 + loc); glBindTexture(GL_TEXTURE_2D, tex)`
     becomes `writeRegs(REG_TEXTURE_<loc>, texParamsAndAddr)`
   - `layout(location = loc) sampler2D tex; vec4 color = texture(tex, texcoord);`
     becomes `vec4 color = sample(REG_TEXTURE_<loc>, texcoord)`
 - Bindless
   - `glUniform1f(loc, val)` becomes `rootTable[UNIFORM_START + loc] = val`
   - `layout(location = loc) uniform float v` in GLSL becomes
     `float v = rootTable[UNIFORM_OFFSET + loc]`
   - `glActiveTexture(GL_TEXTURE0 + loc); glBindTexture(GL_TEXTURE_2D, tex)`
     becomes `memcpy(&rootTable[TEXTURE_OFFSET + TEXTURE_SIZE * loc,
     texParamsAndAddr, TEXTURE_SIZE)`
   - `layout(location = loc) sampler2D tex; vec4 color = texture(tex, texcoord);`
     becomes `vec4 color = sample(&rootTable[TEXTURE_OFFSET + TEXTURE_SIZE *
     loc, texcoord)`
   - It is also possible to have infinite levels of indirection
     `rootTable[N] = secondaryTable`
 - Vulkan
   - `layout(set = S, binding = B) sampler2D textures[]; texture(textures[dynamic_index], texcoord)`
     becomes `sample(&rootTable[S][BINDING_B_OFFSET + TEXTURE_SIZE * dynamic_index], texcoord)`
   - a set is an array of bindings
     - each binding has an array of a descriptor type
     - layout(binding = B) uniform Type[count];

## GART

- GART was introduced with AGP
  - the graphics device is called the master
  - the core-logic (northbridge) is called the target
  - an aperture is a contiguous range of the physical address space where
    master accesses are re-mapped by the target to physically non-contiguous
    system memory pages
  - the remapping is accomplished through GART, Graphics Aperture Re-mapping
    Table, living in the system memory
  - core-logic coherency
    - master accesses outside of the aperture must be coherent
      - master writes is visible to CPU
      - CPU writes is visible to master reads
    - master accesses inside the aperture depends on `ita_coh` and
      `gart_entry_coh`
      - when `gart_entry_coh` is set, `ita_coh` (read-only, core-logic cap)
      	decides the coherency
      - when `gart_entry_coh` is unset, coherency is undefined
- In PCI-e, GART is implemented by the graphics device using the VRAM, not by
  the core-logic
