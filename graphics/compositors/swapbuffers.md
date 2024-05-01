Swap Buffers
============

## Non-fullscreen copying

- app flush
  - `I915_GEM_EXECBUFFER2`
    - event `dma_fence_init`
      - this execbuffer allocates a `i915_request`, which is associated with a
      	`dma_fence`.
    - event `i915_request_queue`
      - we are about to queue the request to the HW ring buffer
    - event `i915_request_add`
      - i915 starts tracking the request status
  - request retiring
    - i915 occasionally checks the status of active requests and retires them
    - when a request is waited for, signaling of the `dma_fence` is enabled
      (i.e., IRQ is enabled) for timley retiring
    - either way, it generates a `dma_fence_signaled` event
- app swap buffers
- X waits for vsync
  - event `drm_vblank_event`
    - this indicates a vblank
- X copies to the window frontbuffer (which is a region of the scanout buffer)
  - `I915_GEM_EXECBUFFER2` to copy
  - `DRM_IOCTL_MODE_DIRTYFB` to dirty the region
    - clflush
  - if GPU copy runs too long, or if GPP rendering for the app takes too long,
    tearing can be seen
- Latency: minimal (presented next vsync after swap buffers)

## Fullscreen flipping

- app swap buffers
- X modesetting driver flips
  - `DRM_IOCTL_MODE_ADDFB2`
  - `DRM_IOCTL_MODE_ATOMIC` -> non-blocking `intel_atomic_commit`
    - `intel_atomic_prepare_commit`
      - calls `intel_prepare_plane_fb` and adds the BO implicit fence that
      	will be waited for
      	* the first time a BO is used for scanout, `intel_plane_pin_fb` calls
      	  `i915_gem_object_pin_to_display_plane` which calls
      	  `i915_gem_object_set_cache_level` which calls
      	  `i915_gem_object_wait`.  The commit becomes blocking.
      	* also in `i915_gem_object_pin_to_display_plane` for the first time,
      	  `__i915_gem_object_flush_for_display` clflushes and adds a fence
      	* they are gone after the first use
    - `drm_atomic_helper_setup_commit` makes sure the previous flip has
      completed, otherwise returns EBUSY
  - `intel_atomic_commit_tail` is called in a wq because of non-blocking
    - it waits the BO first in `intel_atomic_commit_fence_wait`
      - because of the issue in `intel_plane_pin_fb`, it waits for `clflush`
      	worker
    - it waits for the previous flip to complete
      - no-op because of `drm_atomic_helper_setup_commit`
    - it writes to the double-buffered registers
      - event `i915_pipe_update_start`
      - event `intel_update_plane`
      - event `i915_pipe_update_end`
    - it waits for the current flip to complete
      - sends `DRM_EVENT_FLIP_COMPLETE`
      - event `drm_vblank_event`
- Latency: minimal (presented next vsync after swap buffers)
 
## Composited

- app swap buffers
- X waits for vsync
- X wakes up the compositor
- X copies to the window frontbuffer (which is a fake frontbuffer)
  - X copies on vsync even for fake frontbuffer?  This is unnecessary, unless
    it wants to throttle the compositor
- compositor, which is an fullscreen client, swap buffers
  - compositor does not want swap buffers whenever any client has a new buffer
  - it runs on vsyncs; if any client has a new buffer, it recomposites
    everything and swap buffers
  - but there are compositors that simply swap buffers whenever there is any
    damage
- X modesetting driver flips
  - or copies, if the compositor does server-side rendering
- Latency: +one vsync (presented next next vsync after app swap buffers)

## virtio-gpu

- guest app swap bufers
- guest X waits for vsync (no-op because virtio-gpu does not support vsync)
- guest X copies to the scanout buffer and dirty the scanout buffer
  - copy uses glamor, which is GL
  - dirty calls `virtio_gpu_framebuffer_surface_dirty`, which emits a
    `VIRTIO_GPU_CMD_RESOURCE_FLUSH`.
- the hypervisor blits from the guest scanout buffer (which is a GL texture on
  host) to the host window, and does a swap buffer.  The hypervisor is a
  regular client to host X.
- Latency: host swap is minimal (presented next vsync after swap buffers);
  From guest swap to host swap, there are two additional copies
