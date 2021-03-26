# Intel GPUs

## uArchs

- Nehalem
  - Nehalem, 2008, 45nm, 1st gen, Gen4
  - Westmere, 2010, 32nm, 1st gen, Gen5
- Sandy Bridge
  - Sandy Bridge, 2011, 32nm, 2nd gen, Gen6
  - Ivy Bridge, 2012, 22nm, 3th gen, Gen7
- Haswell
  - Haswell, 2013, 22nm, 4th gen, Gen7.5
  - Broadwell, 2014, 14nm, 5th gen, Gen8
- Skylake
  - Skylake, 2015, 14nm, 6th gen, Gen9
  - Kaby Lake, 2016, 14nm, 7th gen, Gen9.5
  - Coffee Lake, 2017, 14nm, 8th/9th gen, Gen9.5
  - (SKIPPED) Cannon Lake, 2018, 10nm, 9th gen, Gen10
  - Comet Lake, 2019, 14nm, 10th gen, Gen9.5
- Ice Lake
  - Ice Lake, 2019, 10nm, 10th gen, Gen11
  - Tiger Lake?, 2020, 10nm, 11th gen, Gen12

## Ice Lake

- a GPU has several engines
  - display
  - media
  - GPE, graphics processing engine
- Gen11LP GT2
  - 1 Unslice, which has
    - 1 GTI
    - 1 Command Streamer
    - 1 Media Fixed Function
    - 1 Blitter
    - 1 Thread Dispatch
    - 1 Geometry
    - 1 POSH
      - tiler
  - 1 slice, which has
    - 1 configurable L3$
      - URB
      - tiler output
      - cache
    - 1 Slice Common
      - 1 Raster
      - 1 HIZ/Z
      - 1 Pixel Dispatch
      - 1 Pixel Backend
    - 8 SubSlices, each has
      - 1 I$ and thread dispatch
      - 1 Sampler
        - Tex$
        - L1$
      - 1 Media Sampler
      - 1 SLM
        - shared local memory
      - 1 Dataport
        - load/store
      - 8 EUs, each has
        - 2 4-wide SIMD ALUs
        - 7 4KB General Register Files, each has
          - 128 GRF registers, each 8x32 bits
          - allow 7 threads
        - 7 Architecture Register Files
        - 1 Thread Control
          - Thread Arbiter picks instructions to run from runnable threads
          - dispatch instructions
          - co-issue up to 4 instructions per cycle
        - 1 Branch unit
        - 1 Send unit

## IRQs

* `gen8_de_irq_handler` for display engine related interrupts
  * `GEN8_PIPE_VBLANK` is generated on vblank, if anyone holds on to vblank
    with `drm_vblank_get`
* `gen8_gt_irq_handler` for graphics related interrupts
  * `GT_RENDER_USER_INTERRUPT` is generated in response to `MI_USER_INTERRUPT` cmd
  * `GT_CONTEXT_SWITCH_INTERRUPT` is generated on context switch
    * always on
    * programmed to be generated at the end of each request
    * requests are submitted to the HW on this interrupt.
    * when GPU is idle, it runs the idle context.  A single request causes
      two context switches: idle->req->idle

## Swap Buffers

### Non-fullscreen copying

* app flush
  * `I915_GEM_EXECBUFFER2`
    * event `dma_fence_init`
      * this execbuffer allocates a `i915_request`, which is associated with a
      	`dma_fence`.
    * event `i915_request_queue`
      * we are about to queue the request to the HW ring buffer
    * event `i915_request_add`
      * i915 starts tracking the request status
  * request retiring
    * i915 occasionally checks the status of active requests and retires them
    * when a request is waited for, signaling of the `dma_fence` is enabled
      (i.e., IRQ is enabled) for timley retiring
    * either way, it generates a `dma_fence_signaled` event
* app swap buffers
* X waits for vsync
  * event `drm_vblank_event`
    * this indicates a vblank
* X copies to the window frontbuffer (which is a region of the scanout buffer)
  * `I915_GEM_EXECBUFFER2` to copy
  * `DRM_IOCTL_MODE_DIRTYFB` to dirty the region
    * clflush
  * if GPU copy runs too long, or if GPP rendering for the app takes too long,
    tearing can be seen
* Latency: minimal (presented next vsync after swap buffers)

### Fullscreen flipping

* app swap buffers
* X modesetting driver flips
  * `DRM_IOCTL_MODE_ADDFB2`
  * `DRM_IOCTL_MODE_ATOMIC` -> non-blocking `intel_atomic_commit`
    * `intel_atomic_prepare_commit`
      * calls `intel_prepare_plane_fb` and adds the BO implicit fence that
      	will be waited for
      	* the first time a BO is used for scanout, `intel_plane_pin_fb` calls
      	  `i915_gem_object_pin_to_display_plane` which calls
      	  `i915_gem_object_set_cache_level` which calls
      	  `i915_gem_object_wait`.  The commit becomes blocking.
      	* also in `i915_gem_object_pin_to_display_plane` for the first time,
      	  `__i915_gem_object_flush_for_display` clflushes and adds a fence
      	* they are gone after the first use
    * `drm_atomic_helper_setup_commit` makes sure the previous flip has
      completed, otherwise returns EBUSY
  * `intel_atomic_commit_tail` is called in a wq because of non-blocking
    * it waits the BO first in `intel_atomic_commit_fence_wait`
      * because of the issue in `intel_plane_pin_fb`, it waits for `clflush`
      	worker
    * it waits for the previous flip to complete
      * no-op because of `drm_atomic_helper_setup_commit`
    * it writes to the double-buffered registers
      * event `i915_pipe_update_start`
      * event `intel_update_plane`
      * event `i915_pipe_update_end`
    * it waits for the current flip to complete
      * sends `DRM_EVENT_FLIP_COMPLETE`
      * event `drm_vblank_event`
* Latency: minimal (presented next vsync after swap buffers)
 
### Composited

* app swap buffers
* X waits for vsync
* X wakes up the compositor
* X copies to the window frontbuffer (which is a fake frontbuffer)
  * X copies on vsync even for fake frontbuffer?  This is unnecessary, unless
    it wants to throttle the compositor
* compositor, which is an fullscreen client, swap buffers
  * compositor does not want swap buffers whenever any client has a new buffer
  * it runs on vsyncs; if any client has a new buffer, it recomposites
    everything and swap buffers
  * but there are compositors that simply swap buffers whenever there is any
    damage
* X modesetting driver flips
  * or copies, if the compositor does server-side rendering
* Latency: +one vsync (presented next next vsync after app swap buffers)

### virtio-gpu

* guest app swap bufers
* guest X waits for vsync (no-op because virtio-gpu does not support vsync)
* guest X copies to the scanout buffer and dirty the scanout buffer
  * copy uses glamor, which is GL
  * dirty calls `virtio_gpu_framebuffer_surface_dirty`, which emits a
    `VIRTIO_GPU_CMD_RESOURCE_FLUSH`.
* the hypervisor blits from the guest scanout buffer (which is a GL texture on
  host) to the host window, and does a swap buffer.  The hypervisor is a
  regular client to host X.
* Latency: host swap is minimal (presented next vsync after swap buffers);
  From guest swap to host swap, there are two additional copies

## Caching

- `i915_gem_object_pin_map` is called when the kernel needs to access the
  BO from CPU
  - e.g., cmd parsing, reloc patching
  - it calls vmap internally to set up a WB or WC mapping
  - this does not modify the kernel linear map
- `i915_gem_mmap_ioctl` is called when ther userspace needs to access the BO
  from CPU
  - it calls `vm_mmap` to set up a vma for the shmem file, and modifies
    `vma->vm_page_prot` to use the desired cache mode

## Implicit Fencing

- in anv, an `anv_bo` is allocated by `anv_device_alloc_bo`
- all VkDeviceMemory are tracked globally
  - `anv_AllocateMemory` adds them to `dev->memory_objects`
- in `anv_queue_execbuf_locked`, `anv_execbuf` is initialized from
  `anv_queue_submit`
  - `anv_execbuf_add_bo` is used to add an `anv_bo` to `anv_execbuf`
  - all but wsi bos are added without `EXEC_OBJECT_WRITE`
  - in `setup_execbuf_for_cmd_buffer`, all internally tracked bos and all
    VkDeviceMemory bos are added to `anv_execbuf`
- in kernel `i915_gem_do_execbuffer`,
  - `eb_lookup_vmas` makes sure each `drm_i915_gem_object` has a `i915_vma`
  - `eb_relocate_parse` calls `eb_validate_vmas`, which calls
    `eb_pin_vma` to pin bo pages inside `i915_vma_pin_ww`
  - `eb_submit` calls `eb_move_to_gpu`, where `i915_vma_move_to_active` calls
    `dma_resv_add_excl_fence` or `dma_resv_add_shared_fence` depending on
    whether `EXEC_OBJECT_WRITE` is set

## ioctls of a simple vk frame

- acquire the frame image
  - `anv_AcquireNextImageKHR`
    - no ioctl
- make sure the frame is "idle" because we reuse frame objects
  - `anv_WaitForFences`
    - `DRM_IOCTL_SYNCOBJ_WAIT`
  - `anv_ResetFences`
    - `DRM_IOCTL_SYNCOBJ_RESET`
- submit the command buffer
  - `anv_QueueSubmit`
    - `DRM_IOCTL_I915_GET_RESET_STATS` to make sure GPU is healthy
    - `DRM_IOCTL_SYNCOBJ_RESET` to reset `pSignalSemaphores` and `fence`
      - for reasons
    - `DRM_IOCTL_I915_GEM_EXECBUFFER2`
- present the frame
  - `anv_QueuePresentKHR`, which calls `wsi_common_queue_present`
    - `anv_WaitForFences` because common wsi reuses frame objects
    - `anv_ResetFences`
    - `anv_QueueSubmit` which is an empty submit with `pWaitSemaphores` from
      `VkPresentInfoKHR` and fence from common wsi
