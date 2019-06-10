# Intel GPUs

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

## Non-fullscreen copying

* app flush
  * `I915_GEM_EXECBUFFER2_WR`
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
* X copies to the scanout buffer
  * event `drm_vblank_event`
    * this indicates a vblank
  * `I915_GEM_EXECBUFFER2_WR` to copy

## Fullscreen flipping

* app swap buffers
* X modesetting flips
  * `DRM_IOCTL_MODE_ADDFB2`
  * `DRM_IOCTL_MODE_ATOMIC` -> non-blocking `intel_atomic_commit`
    * `intel_atomic_prepare_commit`
      * calls `intel_prepare_plane_fb` and adds the BO implicit fence that
      	will be waited for
      	* there appears to be some issues in `intel_plane_pin_fb`.  It calls
      	  `i915_gem_object_pin_to_display_plane` which calls
      	  `i915_gem_object_set_cache_level` which calls
      	  `i915_gem_object_wait`.  The commit becomes blocking.
      	* also in `i915_gem_object_pin_to_display_plane`,
      	  `__i915_gem_object_flush_for_display` clflushes and adds a fence
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
