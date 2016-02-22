Mesa Intel
=================

## intel

* `BATCHBUFFER`, `CMDBUFFER`, `GEM_EXECBUFFER`
* every screen has a bufmgr.  In dri1, sarea and {front,back,depth,tex} are
  also given during init.
* every context has a batchbuffer, a fixed-size bo for commands.  In DRI1 and
  intelInitContext, ctx->{front,back,depth}_region are created from static
  per-screen maps.  A region wraps a bo.
* a intel_framebuffer and abstract renderbuffers are created during intelCreateBuffer.
  every framebuffer has multiple renderbuffers.  A renderbuffer (will) contain a region.
  renderbuffers are made concrete after context binding.

## Fix front-buffer rendering

* <http://bugs.freedesktop.org/show_bug.cgi?id=19174>
* Fixed around 2009/4/10
* Four commits to mesa
  * DRI2: Provide an interface for drivers to flush front-buffer rendering
    * bump `__DRI_DRI2_LOADER_VERSION` to 2
    * require dri2 loader to provide `flushFrontBuffer`: flush fake to real
    * glx x11 loader implements this by calling `dri2WaitGL`, which calls
      `DRI2CopyRegion`, which calls into 2d driver's `CopyRegion`.
    * When there is a fake front buffer, `dri2WaitGL` copies fake one to real
      one; while `dri2WaitX` copies real one to fake one.
  * intel / DRI2: Track and flush front-buffer rendering
    * `glDrawBuffer` specifies which color buffer(s) to draw to
    * new flag `is_front_buffer_rendering` is set when drawing to any front buffer
    * new flag `front_buffer_dirty` is set when drawing to front left buffer.
      It is cleared only when `is_front_buffer_rendering` is set
    * in flush, `flushFrontBuffer` if `front_buffer_dirty`
  * DRI2: Assume that there is always a front buffe
    * server might return only fake front buffer.  It is regarded as a front buffer.
  * intel / DRI2: Accept fake front-buffer from loader
    * `intel_update_renderbuffers` asks loader for buffers.  The returned
      buffers are modeled by `intel_region`.
    * A `gl_framebuffer` has many `gl_renderbuffer`.  rbs and regions are
      associated in `intel_update_renderbuffers`
    * front left rb is special.  It may use real or fake dri front left buffer.
* Three commits to xserver
  * DRI2: Add fake front-buffer to request list for windows
    * For `Window`, insert `DRI2BufferFakeFrontLeft` into `GetBuffers` list if
      a real front buffer is asked and a fake one is not asked
  * DRI2: Do not send the real front buffer of a window to the client
    * Never return `DRI2BufferFrontLeft` in `GetBuffers` if the drawable is a window
  * DRI2: Synchronize the contents of the real and fake front-buffers
    * if there is a fake front buffer, copy contents of real one to the fake one
      in `DRI2GetBuffers`.


## Fake front buffer allocated only when needed

* The goal is

    Front-buffer rendering works, but the fake front-buffer is only
    allocated when it's needed.  JUST LIKE MAGIC!
* Fixed around 2009/4/21.
* Patches against dri2proto, mesa, xserver, intel 2d driver
* dri2proto
  * Add protocol for `DRI2GetBuffersWithFormat`
  * Deprecate old `DRI2GetBuffers`
* mesa
  * DRI2: Implement protocol for `DRI2GetBuffersWithFormat`
    * Implement `DRI2GetBuffersWithFormat` in `src/glx/x11/dri2.c`
  * DRI2: Implement interface for drivers to access `DRI2GetBuffersWithFormat`
    * Bump `__DRI_DRI2_LOADER_VERSION` up to 3
    * require loader to provide `getBuffersWithFormat`
    * With or without format goes through the same code path, but different X
      protocol
    * It is remembered whether the returned buffers have back or fake front
      ones.  It is used for optimization.
  * intel / DRI2: When available, use `DRI2GetBuffersWithFormat`
    * Make `intel_update_renderbuffers` support both interfaces
    * The returned buffer names are compared with current ones.  If the name of
      a renderbuffer is changed, its region is released and a new one is
      allocated for the new name.
* xf86-video-intel
  * DRI2: If the SDK supports it, use the `DRI2GetBuffersWithFormat` interfaces
    * Implement `I830DRI2CreateBuffer` and `I830DRI2DestroyBuffer`, both
      singular.
    * When the buffer asked is not front left, a new Pixmap is created.
      Otherwise, the drawable itself is used.
    * The pixmap has a bo associated with it.  It is the bo's name that is
      returned.
* xserver
  * DRI2: Implement protocol for `DRI2GetBuffersWithFormat`
    * `glx/glxdri2.c` is modified because it is a loader for `AIGLX`
    * `DRI2QueryVersion` reports 1.1.
    * The major function is `do_get_buffers`.  For each buffer asked, if the
      same attachment is already available (exist and same w/h), it is reused.
      Otherwise, a new buffer is created.  Any existing buffer that is not
      requested nor reused at this round is destroyed.  Previously, existing
      buffers were destroyed and a new set of buffers were allocated.  There was
      no reuse.
    * When only back buffer is asked, both real front and back buffers are
      allocated;  When only front buffer is asked, both real and fake front
      buffers are allocated;  That is, double buffering does not need a fake
      front buffer.
    * Single buffer rendering does not need a fake front buffer when the target
      is a pixmap (or pbuffer).

## Screen Configs

* Initialized in `intelInitScreen2`
* Three `fb_format` and `fb_type`
  * `GL_RGB`: `GL_UNSIGNED_SHORT_5_6_5`
  * `GL_BGR`: `GL_UNSIGNED_INT_8_8_8_8_REV`
  * `GL_BGRA`: `GL_UNSIGNED_INT_8_8_8_8_REV`
* `back_buffer_modes = GLX_NONE, GLX_SWAP_UNDEFINED_OML, GLX_SWAP_COPY_OML`
* (depth, stencil)
  * RGB565: (0, 0), (16, 0)
  * others: (0, 0), (24, 0), (24, 8)
* For each format and type, `driCreateConfigs` is called to create configs.
  * `num_modes = num_depth_stencil_bits * num_db_modes * num_accum_bits *
     num_msaa_modes`
  * E.g., RGB565 creates 2 x 3 x 2 x 1 = 12 modes
  * having accum buffer implies slow config `visualRating`
  * `db_mode` GLX_NONE is single buffered; otherwise, double buffered.
  * depth or stencil buffer having zero bits means no such buffer

## Double Buffering

* Back left render buffer is created only when double buffering.
* As a result, `__DRI_BUFFER_BACK_LEFT` is asked only when double buffering
* `intelSwapBuffers` does nothing but throttle when not double buffering
* Otherwise, it waits for vsync and copy back buffer to front one.

## COW

* A `struct intel_buffer_object` is a VBO/PBO.  It might use a GEM buffer or
  system buffer.
  * when data are submitted, depending on the target, it might use GEM buffer or
    malloc system buffer to memcpy the passed data.
  * In case of system buffer, mapping/unmapping is a no-op.
  * `intel_bufferobj_buffer` asks for the GEM buffer of a buffer object.  If
    it is attached to a region, it is detached.  If it is system buffer based,
    it is migrated to GEM based.
* A `struct intel_region` wraps a GEM buffer or a PBO.
  * `intel_region_attach_pbo` attaches a pbo to a region.  It unrefs the
    region's GEM buffer and makes it uses PBO's.
  * `intel_region_release_pbo` releases the attached pbo and allocates a new GEM
    buffer.
  * `intel_region_cow` releases the attached pbo, and copies the contents of the
    pbo to the new gem buffer.
  * 

## State

* hw state is tracked by `struct i915_hw_state`
  * `active` records the enabled units.
  * `emitted` records the units that already have the init code in the batch.
  * to get a list of units to be init, `active & ~emitted`.
  * `vtbl.new_batch` sets `emitted` to zero, because new batch is started.
  * all changes to state is buffered until `vtbl.emit_state`.
* `I915_STATECHANGE` is used to indicate certain state is about to be changed;
  `I915_ACTIVESTATE` is used to indicate certain state is about to be enabled.
* When a context is created, `i915InitStateFunctions` is called to install state
  related driver functions.
* Before context creation returns, it calls `i915InitState` to initialize the
  state.
  * `i915_init_packets` is called to initialize the state and to mark the active
    units.
  * `_mesa_init_driver_state` is called to call various GL functions so that the
    state of `struct GLcontext` matches the hw state.
  * the initial state is remembered in `initial` hw state.
* every batch is atomic, when talking about context switch.  It usually starts
  with state programming instructions followed by rasterization.
* hw state is reprogrammed through `i915_emit_state`
  * `intel->batch`, `state->draw_region`, `state->depth_region`
  * `state->tex_buffer[]` (there are 8 on i915)
  * drawing regions are changed when `glDrawBuffer`, etc.
  * `tex_buffer[]` is changed when running pipeline.  `intelRunPipeline` checks
    if `_NEW_TEXTURE` is set and it runs `vtbl.update_texture_state` beforehand.
    For each `_ReallyEnabled` texture unit, `i915_update_tex_unit` is called to
    program the state of the unit.

## Flush

* `vtbl` is hw-specific.
* `_intel_batchbuffer_flush`
  * `intel->vtbl.flush_cmd` to add flush cmd to the end of the batch.
  * `intel->vtbl.finish_batch` to submit `vb` to `vb_bo`.
  * `do_flush_locked`
    * unmap and exec the batchbuffer
    * `intel->vtbl.new_batch`

## Intel Buffer Object

* `struct intel_buffer_object` is derived from `struct gl_buffer_object`.
* It (or its base class)
  * has a `Name`, which might be 0.
  * is initially empty (no backing store).
  * has backing store from `sys_buffer` or dri `buffer`.
  * might have its dri `buffer` from the attached `struct intel_region`.
* Mapping
  * It returns `sys_buffer` directly if that is what it bases on.
  * Or, it returns GTT/normal mmaped dri `buffer`, depending on
    `kernel_exec_fencing`.
  * Or, NULL if it is empty.
* `BufferData`
  * asserts the object is unmapped
  * empties the object (destroy current buffer if any)
    * contrary to `_mesa_buffer_data`, which `realloc`.
  * if there are new data, allocates a new buffer for them
    * Actually, if size is given, a new buffer is allocated.  If data are also
      given, they are copied to the new buffer.
    * It is not uncommon that `BufferData` is called to allocate only a buffer
      and `MapBuffer` is called to fill the buffer.
* `BufferSubData`
  * copies the data to the buffer at the given offset
  * no buffer is allocated or destroyed

## Fallbacking

* When switching to fallback mode in `intelFallback` (`mode` is true for any
  `bit`), `intelFlush` is called to flush hw, and `_swsetup_Wakeup` is called
  to hook swrast to the render stage. 
  * `intel->Fallback` is non-zero
* When switching to hw mode, `_swrast_flush` is called to flush swrast, and
  hw render functions are hooked to the render stage.
  * These functions call into `vtbl`.
* `intelRunPipeline` is called to run the pipeline
  * If fallbacking, `_swsetup_Wakeup` has already took care of everything
  * If hw, `intelChooseRenderState` is called to choose the tri-functions
  * After setting up, `_tnl_run_pipeline` is called.
* Tri-functions
  * For fallbacking, they are generated from `ss_tritmp.h`.  After some vertex
    processing, it calls `_swrast_<Prim>` for rasterization.
  * For hw, they are generated from `tnl_dd/t_dd_tritmp.h`.  After some vertex
    processing, it calls `<PRIM>` for rasterization.  Sometimes, a mix of
    accel. and unaccel. primitives are being drawn.  `<PRIM>` calls
    `intel_draw_<prim>` or `intel_fallback_<prim>` respectively.
    * In the latter case, it falls back to swrast.
    * are they used?
* The render stage of the pipeline
  * it is the last stage, `_tnl_render_stage`
  * it may be hijacked by `intel_run_render`, see `intel_pipeline`
  * i915 always sets `_MaintainTexEnvProgram` to use frag prog to replace
    fixed-functions.
  * it emits vertices in hw format to a VBO, or the batchbuffer for inlining
    with the commands


## BO

* batch buffer
  * no tiling
  * instructions are written to system memory and DMA into the bo
* VBO
  * no tiling
  * mapped gtt if write-only
  * otherwise, mapped cpu
* RBO
  * no tiling unless depth rbo on sandybridge (requirement)
  * `BO_ALLOC_FOR_RENDER` is set
  * mapped gtt for swrast
* texture
  * color texture has tiling x when the width is greater than 64
  * depth texture has tiling y
  * `BO_ALLOC_FOR_RENDER` is set if render-to-texture is expected
  * width align is 4, height align is 2 (only when computing the size)
  * for both swrast sampling or image speficiation, mapped gtt if tiled;
    otherwise, mapped cpu
* scan out
  * `i830_allocate_framebuffer` in xorg-video-intel
  * prefers tiling X
  * width must align to 64 (only for computing the buffer size)
* tiling
  * when tiled, must map gtt
  * `I915_GEM_SET_TILING` requires both tiling and stride
    * the kernel checks if tiling is possible with the given stride
  * i915g
    * for 32-bit scan out with width greater than 64, tiled X and the stride is
      made power of two.
    * for 32-bit shared texture with width greater than 240, tiled X and te
      stride is made power of two.
    * texture is created with the given stride and height, then the tiling is
      set.
      * when the stride is not properly aligned, setting the filing fails.
        That's why the mis-rendering is "fixed" because there is no tiling at
        all.
    * for texture from handle, the stride is given and the tiling is got by
      querying the bo
  * i915c
    * `intel_region::pitch` is in pixels
    * `__DRIbufferRec::pitch` is in bytes
    * `createImageFromName` is in bytes
      * by definition.  But the implementation uses pixels.
    * `intel_region_alloc_for_handle` uses the stride given and the tiling
      queried
  * possible issues
    * i915g does not error check set_tiling
    * 

* Android
  * activity renders with cpu unless 3d games with hw GL
  * surfaceflinger renders to scan-out buffer with GL
  * surfaceflinger samples activity buffers with GL
  * scan-out buffer should be mapped gtt
  * activity buffer
    * renders and samples a lot
    * tiling?
    * mapping?

## Kernel

* GEM objects are created with `i915_gem_alloc_object`
  * it calls `drm_gem_object_init` to create a shmem object
  * pre-gen6, the object is marked `I915_CACHE_NONE`
  * when `DRM_IOCTL_I915_GEM_MMAP` is called, the shmem is mmap()ed.  The
    mapping uses CPU MMU and is cached.
  * when `DRM_IOCTL_I915_GEM_MMAP_GTT` is called, the pages of the shmem are
    paged in in `i915_gem_fault`.  It indirectly calls `intel_gtt_insert_pages`
    and the mapping uses AGP aperture.  Because the object is `I915_CACHE_NONE`,
    it is uncached (write-combined?).
