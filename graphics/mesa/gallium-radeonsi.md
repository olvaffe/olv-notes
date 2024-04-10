Gallium Radeon SI
=================

## `GL_RENDERER`

- e.g., `AMD Radeon Graphics (renoir, LLVM 16.0.6, DRM 3.54, 6.5.4)`
- `si_init_renderer_string`
  - `first_name (second_name, LLVM version, DRM version, kernel_version)`
  - `first_name` is `radeon_info::marketing_name` or `radeon_info::name`
    - `marketing_name` is `amdgpu_get_marketing_name`
    - `name` is `enum radeon_family` with `CHIP_` prefix stripped
  - `second_name` is `radeon_info::lowercase_name`
    - this is `radeon_info:name` in lower csae
  - LLVM version is `MESA_LLVM_VERSION_STRING`
  - DRM version is `radeon_info::drm_major` and `radeon_info::drm_minor`
  - `kernel_version` is `utsname::release`
- radv
  - `VkPhysicalDeviceProperties::deviceName` is
    `radv_physical_device::marketing_name`
  - `%s (RADV %s%s)`
    - first string is `amdgpu_get_marketing_name` or `AMD Unknown`
    - second string is `radeon_info::name`
    - third string is `radv_get_compiler_string` and is empty unless llvm is
      used

## winsys

- `amdgpu_winsys_create` creates the winsys
  - call stack
    - `amdgpu_winsys_create`
    - `radeonsi_screen_create`
    - `pipe_radeonsi_create_screen`
    - `pipe_loader_drm_create_screen`
  - the function calls back to `radeonsi_screen_create_impl` to create the
    screen
- `amdgpu_screen_winsys`, `amdgpu_winsys`, and `amdgpu_device_handle`
  - `amdgpu_device_handle` wraps a kernel `struct drm_device`
    - when `amdgpu_device_initialize` is called twice, and the fds refer to
      the same kernel `struct drm_device`, the same `amdgpu_device_handle` is
      returned
    - when the two fds refer to the same kernel `struct drm_device` but
      different kernel `struct drm_file`, things get tricky
      - `amdgpu_device_handle` only uses the first fd
      - the second fd is completely ignored
  - `amdgpu_winsys` wraps a `amdgpu_device_handle`
    - `dev_tab` and `dev_tab_mutex` make sure there is a 1:1 mapping
    - `amdgpu_winsys::fd` and `amdgpu_device_get_fd()` always refer to the
      same `struct drm_file`
  - `amdgpu_screen_winsys` wraps a `struct drm_file`
    - when `amdgpu_winsys_create` is called twice, and the fds refer to the
      same kernel `struct drm_file`, the same `amdgpu_screen_winsys` is
      returned
  - `are_file_descriptions_equal` returns true when two fds refer to the same
    `struct drm_file`
- `amdgpu_bo_real` and `amdgpu_bo_handle`
  - `amdgpu_bo_handle` wraps a kernel `struct drm_gem_object`
    - `amdgpu_device_handle` makes sure there is a 1:1 mapping
      - when `amdgpu_bo_alloc` allocates a new bo, the userspace creates a new
        `amdgpu_bo_handle` and the kernel creates a new `drm_gem_object`
      - when `amdgpu_bo_import` imports a dma-buf, the userspace creates a new
        `amdgpu_bo_handle` only when the kernel creates a new `drm_gem_object`
  - `amdgpu_bo_real` wraps a `amdgpu_bo_handle`
- when a client opens a render node and passes the fd to `gbm_create_device`
  - if there is no pre-existing `amdgpu_device_handle` for the render node,
    this
    - creates `amdgpu_device_handle` with the fd
    - creates `amdgpu_winsys` with the fd
    - creates `amdgpu_screen_winsys` with the fd
    - `gbm_bo_create` allocates the bo using the fd
  - but if there is a pre-existing `amdgpu_device_handle` for the render node,
    because the fd refers to a different kernel `struct drm_file`, it
    - reuses `amdgpu_device_handle` with a different fd
    - reuses or creates `amdgpu_winsys`, depending on the pre-xisting
      `amdgpu_device_handle` is created by `amdgpu_winsys_create` or not
    - creates `amdgpu_screen_winsys` with the fd
    - `gbm_bo_create` allocates the bo using the different fd
      - then exported as dma-buf and imported into this fd
- `eglInitialize`, `gbm_create_device`, and `vaInitialize`
  - all three call `radeonsi_screen_create`
    - `amdgpu_winsys_create` calls `radeonsi_screen_create_impl`
    - `radeonsi_screen_create_impl` calls `si_create_context` twice for
      `aux_contexts`
  - when they are all called for the same render node from the same process
    - the first `amdgpu_winsys_create` call
      - creates a new `amdgpu_winsys`
      - creates a new `amdgpu_screen_winsys`
      - creates a new `si_screen`
    - the rest `amdgpu_winsys_create` calls
      - reuse the same `amdgpu_winsys`
      - create new `amdgpu_winsys`
      - creates new `si_screen`
      - `are_file_descriptions_equal` should be false
    - but if `amdgpu_winsys_create` is made a private symbol,
      - `eglInitialize` and `gbm_create_device` both load `radeonsi_dri.so`
      - `vaInitialize` loads `radeonsi_drv_video.so`
      - the first `amdgpu_winsys_create` call behaves the same as before
      - the second `amdgpu_winsys_create` call, if called from a different
        library,
        - creates a new `amdgpu_winsys` (instead of reusing)
        - creates a new `amdgpu_screen_winsys`
        - creates a new `si_screen`

## Flush

- `glFlush` dispatches to `_mesa_Flush`
  - `PIPE_FLUSH_ASYNC` is set unless the gl context uses egl images and
    `ctx->Shared->HasExternallySharedImages` is true
- `eglSwapBuffers` dispatches to `dri2_swap_buffers`
  - all platforms call `dri2_flush_drawable_for_swapbuffers` to flush
  - that jumps to `dri_flush` in the DRI driver with `__DRI2_FLUSH_DRAWABLE`
    and `__DRI2_FLUSH_INVALIDATE_ANCILLARY` as the flags and
    `__DRI2_THROTTLE_SWAPBUFFER` as the reason
  - `st_context_flush` is called `ST_FLUSH_END_OF_FRAME`
    - it translate the flag to `PIPE_FLUSH_END_OF_FRAME`
- `si_flush_from_st` always calls `si_flush_gfx_cs` with `PIPE_FLUSH_ASYNC`
  - `RADEON_FLUSH_START_NEXT_GFX_IB_NOW` is set on newer kernels
- `amdgpu_cs_flush` adds the submit to the queue
  - because `PIPE_FLUSH_ASYNC` is set, it happens some time later

## glxgears and threads

- `glxgears` relies on `glXSwapBuffers` to flush
- `glXSwapBuffers` calls `dri_flush` in st/dri
  - because we are about the share the image with the x server, we need to
    flush all userspace commands so that the image will be implicitly fenced
    by the kernel space
  - the flush is done by `_mesa_glthread_finish` followed by
    `st_context_flush`
- `_mesa_glthread_finish` waits for the `gl` thread to become idle and
  executes all GL commands that have been recorded to the current batch
  - because we did not submit any batch to the `gl` thread, there is no need
    to wait
- `st_context_flush` calls `pipe->flush` which points to `tc_flush`
  - the flush flag has only `PIPE_FLUSH_END_OF_FRAME` and the flush is
    executed directly
  - `tc_sync_msg` waits for the `gdrv` first, and because there is no
    submitted batches, there is no need to wait
  - `si_flush_from_st` is then called
- `si_flush_from_st`
  - `si_flush_all_queues` calls `si_flush_gfx_cs` which calls `amdgpu_cs_flush`
    - because `PIPE_FLUSH_ASYNC` is always set, `amdgpu_cs_flush` submits the
      job to the `cs` thread
    - the `cs` thread calls `amdgpu_cs_submit_ib` to submit the job to the
      kernel driver
  - it also call `amdgpu_cs_sync_flush` to wait for the submit
