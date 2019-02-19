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
