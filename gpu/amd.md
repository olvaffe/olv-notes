# AMD

## GCN

* Micro Engine / Micro Engine Compute, ME / MEC
  * there is a ME, aka graphics command processor, aka CP
    * only one queue?  thus the kernel assigns each process a software queue,
      and copies commands from the SW queue to the single HW queue, to avoid
      one process taking over the queue
    * the queue supports DRAW and COMPUTE commands
    * in the old days, ME had one graphics queue and two compute queues
  * there are one or two (or more) MECs
    * each MEC has 4 threads, aka pipes, aka ACEs (asyn compute engine)
    * each pipe has 8 compute queues, aka compute rings
    * e.g., the kernel exposes only some of the queues to the userspace
      * hw with 1 MEC: all 8 queues in the first pipe
      * hw with 2 MECs: first 2 queues of each of the 4 pipes in the first MEC
    * the queues support only COMPUTE commands
* Hardware Scheduler, HWS
  * dispatch tasks from ME/MECs to CUs?
* Shader Engine, SE
  * a SE consists of 1 geometry processor (1 primitive/cycle), 1 rasterizer, 1
    to 16 CUs, 1 to 4 Render Back Ends (RBEs); I think a SE is one
    fixed-function graphics pipeline.
  * a GPU normally has 1 to 4 SEs; each SE has `max_sh_per_se * max_cu_per_sh`
    CUs; on higher-end GPUs, the total number of CUs is in the 64 ballpark
* Compute Unit, CU
  * a CU consists of a CU scheduler, a branch&message unit, 4 SIMD-VUs, 4
    64KiB register arrays, and 4 Texture Filter Units.
  * CU scheduler is different from HWS
    * it groups threads into wavefronts and assigns wavefronts to SIMD-VUs
* SIMD Vector Unit, SIMD-VU
  * 256 vector registers, each consists of 64 floats (64KiB in total)
  * a wavefront consists of 64 threads, with thread N using channel N of the
    registers
  * can execute up to 10 wavefronts at the same time, depending on how many
    registers a thread needs
    * e.g., when a fragment shader needs 32 registers, there can be 8
      wavefronts processing 512 fragments at the same time
  * has 16-lane ALU; thus it takes 4 cycles to execute one wavefront
