Mesa and Its Main Context
=========================

## Mesa Code Flow

- From `mesa-glapi`, we know how user calls reach mesa functions.
- There is a main context `struct GLcontext`.  There are several module
  contexts, like vbo, tnl, shader, in the main context, to support different
  functions of OpenGL.  It basically works like
  1. contexts' states are changed by user
  2. vertices from user enter mesa
  3. flush, and vertices are passed to tnl to run the pipeline
  4. the last stage of the pipeline is rendering
- Laziness
  - states are updated lazily in `_mesa_update_state`.
  - Vertices are buffered in vbo context.  They are flushed to tnl context for
    `draw_prims`, which might be buffered again in the hw or swrast context.
  - `glFlush` to make sure they reach the hw.

## States

- `struct GLcontext` consists of many components.  Whenever the user changes any
  of the components, mesa makes sure the buffered data are flushed before any
  change.  The changes are made in a half-lazy way.  It only checks the
  validities of the changes and does the minimal changes to the context.
  `NewState` is marked for updates and when `_mesa_update_state` is called at
  some point in the future, the changes are finally reflected by each
  components.
- Some changes are made to the components directly.  When flushing, they
  propagate back to the main context.
  - `glColor` changes vbo directly.  They will be copied back to main context
    when flushing.
- Some components of `struct GLcontext` are chosen by the driver (those context
  members of type `void *`).  At the end of `_mesa_update_state`,
  `ctx->Driver.UpdateState` is called to let the driver update its and those
  components' states.
- Take a look at `i915`.  `i915InvalidateState` is called when updating the
  states.  Components like `swrast`, `swrast_setup`, `vbo`, and `tnl` are
  updated.
- There are 3 places the states are stored and should be synchronized. 
  - MESA, the states are updated by user
  - DRIVER, the states are updated by MESA when `InvalidateState`.
  - HW, the states are updated by DRIVER.
- When a state in MESA is changed,
  - if it is used by bufferred vertices, it should `FLUSH_VERTICES`
  - if it uses another state, it should `FLUSH_CURRENT` or `_mesa_update_state`.

## Conventions

- `Enable` means the function is enabled by `glEnable`, etc.
  `_Enable` means the function will be used.  E.g., an enabled program without a
  bound program cannot be used.
- `Current` means the current bound object.
  `_Current` means the object that will be used.  There might be no bound object
  but there might be a default object.

## Attribute Groups

- Every attr groups is a memeber in the main context
  - group `GL_XXX_BIT` usually corresponds to `ctx->Xxx` of type
    `struct gl_xxx_attrib`.
- Some `struct gl_xxx_attrib` are not attr groups that can be pushed.
- `GL_COLOR_BUFFER_BIT`
  - one of the attribute is which renderbuffers to draw.  With the introduction
    of FBO, the real draw buffers are set in the FBO `ctx->DrawBuffer`.
    `ctx->Color.DrawBuffer` is used to remember the winsys fbo's draw buffers.
    See also `_mesa_drawbuffers`.
- `GL_ENABLE_BIT`
  - There is a local `struct gl_enable_attrib`, and main context does not have a
    memeber of that type.
  - When pushed, the enable bits are derived from every place of the main
    context, and are stored in `struct gl_enable_attrib`.
- `GL_PIXEL_MODE_BIT`
  - The read buffer is from `ctx->ReadBuffer->ColorReadBuffer`.  See also
    `_mesa_readbuffer`.
- `GL_TEXTURE_BIT`
  - `ctx->Texture` is of type `struct gl_texture_attrib`.  But when pushed,
    `struct texture_state` is pushed.  It is to hold both the attrib and all
    current texture objects.
- `GL_CLIENT_PIXEL_STORE_BIT`
  - It pushes `ctx->Pack` and `ctx->Unpack`.  It affects the the data
    format/source of `glDrawPixels`, `glReadPixels`, texture, and some more
    functions.
- `GL_CLIENT_VERTEX_ARRAY_BIT`
  - It pushes `ctx->Array`.  In `struct gl_array_attrib`, there is
    `struct gl_array_object` for VAO and `struct gl_buffer_object` for VBO.

## Q & A

- what would happen if two apps doing DRI at the same time?  context switch?
  - context is resent every batch.  see i915_new_batch and i915_emit_state.
     HARDWARE_LOCKING?
- In DRI2, clients draw directly to front or back buffer.  front buffer is
  window's pixmap.  it is offscreen when composite is enabled.  is this the
  reason DRI2 supports redirected direct rendering?  why does DRI1 not support
  it?
  - Yes(?).  DRI1 does not support it because DRI drmMap screen buffer's handle.
  which is always on-screen.
- how to support `EXT_framebuffer_blit`?  no `glBlitFramebufferEXT` symbol?
  - first, `glBlitFramebufferEXT` does not have static dispatch.
     This is true for extension functions.  extension functions are queried at runtime.
     To create a dynamic dispatch, in `intel_extensions.c`,
     `need_GL_EXT_framebuffer_blit` is defined and `extension_helper.h` is included.
     `intel->ctx.Driver.BlitFramebuffer = intel_blit_framebuffer`

## FBO, VBO, PBO

- VBO is represented by `struct gl_buffer_object`.  See `_mesa_GenBuffersARB`.
  The `Name` is an unque id dynamically generated.
- Binding a buffer updates
  - `ctx->Array.ArrayBufferObj`, when binding to `GL_ARRAY_BUFFER_ARB`.
  - `ctx->Array.ElementArrayBufferObj`, when binding to `GL_ELEMENT_ARRAY_BUFFER_ARB`.
  - ...
- PBO is, api wise, VBO plus two binding targets.
  - `GL_PIXEL_PACK_BUFFER_EXT` binds to `ctx->Pack.BufferObj`
  - `GL_PIXEL_UNPACK_BUFFER_EXT` binds to `ctx->Unpack.BufferObj`.
  - When a PBO is bound, it affects functions like `glReadPixels`,
    `glGetTexImage`, `glDrawPixels`, `glTexImage2D` to source from the bound
    PBO.
- A `struct GLcontext` has its `{WinSys,}{Draw,Read}Buffer` pointing to NULL
  initially.
  - FBO is represented by `struct gl_framebuffer`.
  - `_mesa_make_current` updates `WinSys{Draw,Read}Buffer`.  The given
    framebuffers should not have names, that is, not user-generated.
  - `{Draw,Read}Buffer` points to either user-generated or winsys FBOs.

## VAO

- Yeah, `gl_array_object`, vertex array object.  It consists of many client
  arrays.
  - conventional arrays: vertex, normal, color, ...
    - There are 8 texture arrays.  It makes a total 16 conventional arrays.
  - 16 or so generic attrib arrays, introduced for vertex program.
  - conventional and generic arrays are separated.  Functions for convertional
    arrays do not work on generic arrays, and vice versa.
- By default, VAO zero is bound.
- To enable arrays,
  - `glEnableClientState` to enable conventional arrays
    - There are multiple texture coord arrays.  They are selected by
      `glClientActiveTexture`.
  - `glEnableVertexAttribArray` to enable attrib arrays
- The pointers of arrays can be set by
  - `glXxxPointer` for conventional arrays
    - Again, `glTexCoordPointer` is affected by `glClientActiveTexture`.
  - `glVertexAttribPointer` for generic arrays.

## Textures

- A context has many texture units.  A unit has many texture targets, but only
  one is complete and enabled at any given time.  A target is bound to one
  texture object at any given time.  A texture object has one texture image for
  each level (mipmaps).
  - current text unit is `&ctx->Texture.Unit[ctx->Texture.CurrentUnit]`.
  - texture images are `texObj->Image[MAX_FACES][MAX_TEXTURE_LEVELS]`.
- This setup is to accomplish
  - Texture objects are created by user.  They have bound targets (1D, 2D, etc.).
  - Every texture unit can be enabled and bound with a texture object at a time.
  - The color of a fragment is decided by repeatedly combining texels and output
    of previous texture unit until texture unit 0.
- Multiple texture units are defined by `ARB_multitexture`.
  - `glTexCoord` is always for unit 0.  Not affected by `glActiveTexture`.
  - `glMultiTexCoord` is for any unit.
  - `glTexCoordPointer` is affected by `glClientActiveTexture`.
- `glTexEnv` sets params of current texture unit.
- `glGenTextures` generates unique ids and new `struct gl_texture_object` by
  `ctx->Driver.NewTextureObject`.
- `glBindTexture` binds the texture to the given target of current unit.
- `glTexParameter` sets params of the given target of current unit.
- `glTexImage2D` remembers the given info in the texture image and calls
  `ctx->Driver.TexImage2D`.  Driver should upload the pixels to a suitable
  place, using memcpy, hw, or whatever.
  - Pixels might be uploaded/unpacked from a PBO.  In that case, pixels pointer
    is actually an offset into the PBO.
- When a texture object is attached to an FBO, `ctx->Driver.RenderTexture` is
  called.  It usually creates an rb wrapping the texture object and sets
  attachment's `Renderbuffer` to it.
- Texture object parameters are changed by `glTexParameter`
  - `TEXTURE_MIN_LOD` decides `lod_min`, -1000 by default
  - `TEXTURE_MAX_LOD` decides `lod_max`, 1000 by default
  - `TEXTURE_BASE_LEVEL` decides `level_base`, 0 by default
  - `TEXTURE_MAX_LEVEL` decides `level_max`, 1000 by default
  - A lambda is calculated.  Usually, minification is used when lambda is larger
    than 0; magnification if lambda is smaller than 0.  lambda is clamped by
    `lod_min` and `lod_max`.
  - When no mipmapping, texture image at `level_base` is used.  Mipmapping is
    only for minification so magification always use `level_base`.
  - When the image at `level_base` is changed, and automatic mipmap generation
    is enabled, images from `level_base` to `min(computed-max-level, level_max)`
    is generated.
  - A texture object is complete if
    - the set of mipmap arrays has the same internal format and has needed
      width and height.
    - borders of images are the same
    - `level_base` has width, height greater than 0.
  - Mipmap must be used with complete texture object.

## FBO and `GL_EXT_framebuffer_object`

- FBO is represented by `struct gl_framebuffer`.
  - `glGenFramebuffersEXT` to generate, `glBindFramebufferEXT` to bind.
  - It is created at binding time by `ctx->Driver.NewFramebuffer`.
  - `ColorDrawBuffer`, `ColorReadBuffer` gives current drawing/reading buffers
    in `GL_COLOR_ATTACHMENT0_EXT`, `GL_BACK`, etc.
  - Binding makes `ctx->{Draw,Read}Buffer` pointing to the new FBOs.
- One of the attachments is `strut gl_renderbuffer`.
  - `glGenRenderbuffersEXT` to generate, `glBindRenderbufferEXT` to bind.
  - It is created at binding time by `ctx->Driver.NewRenderbuffer`. 
  - Binding makes `ctx->CurrentRenderbuffer` pointing to the new rb.
  - It is bound so that storage can be specified by `glRenderbufferStorageEXT`.
- The other of the attachments is `struct gl_texture_object`.
- To attach an attachment to an FBO, one must give
  - `target` to specify which of `ctx->{Draw,Read}Buffer` to be used.
  - `attachmentPoint` to specify where to attach to.
  - attachment id and its specific parameters.
  - `glFramebufferTexture2DEXT` and `glFramebufferRenderbufferEXT`.

## FBO and Winsys

- An fbo has a list of attachment points.  Both rb and tex can be attached.
- An fbo has read buffer and draw buffer(s).  One may specify which attachment
  should the fbo draw to or read from.
- Winsys fbo is initialized by `_mesa_initialize_framebuffer`.
- RBs are initialized by `_mesa_init_renderbuffer` and attached by
  `_mesa_add_renderbuffer`.
  - rb is added to `fb->Attachment[bufferName]`.
  - the type is `GL_RENDERBUFFER_EXT`.
- Winsys fbos are bound to the context by `_mesa_make_current`.  They are bound
  to `ctx->Winsys{Draw,Read}Buffer`.

## Extension

- Take `ARB_framebuffer_object` for example
- Add `GLboolean ARB_framebuffer_object` to `struct gl_extensions`.
- Add an entry to `default_extensions` in `main/extensions.c`.
  - Default values listed here are copied to `ctx->Extensions` on
    `_mesa_init_extensions`.
  - Individual values are changed by `_mesa_enable_extension` or convenient
    functions.
- Let driver enable it
  - Add hooks to `ctx.Driver` for the functions needed by this extension
  - Use driver-specific way to enable the extension.  This usually calls
    `_mesa_enable_extension` and add dispatches to those not statically
    dispatched.


## Texture Object (sec. 3.8)

- A texture object stores two categories of states
  - Those in `struct gl_texture_object`, modified by `glParameter`.
  - Those in `struct gl_texture_image`.
- A texture image is null if it has zero width, height, depth, and border.  It
  should also have internal format `1`, zero compress, zero-sized components.
- level 0 image is called the main texture image.  It is the only image that
  guarantees to support width and height of `GL_MAX_TEXTURE_SIZE`.
- level base is the level used for sampling, unless mipmap is seleted.
- A mipmap is the texture images from base level to 1x1.  See 3.8.8.
- The spec says two things
  - If the base level has a null texture image, it is as if texturing is
    disabled.
  - If the texture is not complete, and mipmap is used, it is as if texturing is
    disabled.
  - `_mesa_test_texobj_completeness` changes the definition of complete to unify
    the above 2 statements.
- A texture image have two operations
  - FetchTexel: given x, y, z, return the sample
  - StoreImage: store samples into an image
- By default, software fallback is used
  - FetchTexel fetches texel from `texImage->Data`.
  - StoreImage stores texels in `texImage->Data`.
- For hw implementation, `texImage->Data` is usually `NULL`.
  - Take a look at `intelTexImage`.  In case there is no PBO, the buffer is
    mapped and the software fallback is called to store the texels.  The buffer
    is unmapped imeediately after storing.  Most of the time, `texImage->Data`
    is `NULL`.
  - Rendering in intel driver is usually pure hw.  But in case a fallback is
    needed, the mipmap tree is temporarily mapped so that the sw fallback can do
    its work.
- Another thing that is special in hw is
  - All texture images might share a single buffer.  There is usually a hw
    restriction that a texture object corresponds to a single buffer.  The
    mipmap set should store their data in that single buffer.


## Mesa Formats

- Given the format and type of the data, the pixel storage/transfer operations
  convert the data into RGBA, clampped
  - Unless for texture image, they are then converted to `GLchan`, the
    fixed-point representation
- In texture image packing, the data are
  - converted to RGBA, clampped.  The conversion depends on `format` and
    `type`.  See `extract_float_rgba`.
  - then converted to base format of `internalFormat`.
- The base formats describe mainly the components required
  - the real internal format for `GL_RGBA` might be
    - `MESA_FORMAT_RGBA8888`
    - `MESA_FORMAT_RGBA8888_REV`
    - `MESA_FORMAT_RGBA5551`
    - `MESA_FORMAT_ARGB8888`
    - and many more
- Mesa formats are in host-endian.
  - the memory layout of `MESA_FORMAT_RGBA8888` on big-endian machines
    are `R, G, B, A`, from low memory to high memory
  - the memory layout of `MESA_FORMAT_RGBA8888` on little-endian machines
    are `A, B, G, R`, from low memory to high memory
- In `TexImage2D`, a driver should
  - decide the mesa format from the internal format
  - call the mesa format's `StoreImage`
  - take argb8888 for example
    - the user data are converted to RGBA GLchan first
    - the results are converted again to `(a << 24 | r << 16 | g << 8 | b)`
      where the a,r,g,b are converted from GLchan to ubyte.
- Android `PIXEL_FORMAT_RGBA_8888` is in memory `R,G,B,A`, pre-multiplied
  - When packed in `uint32_t`, it is host-endian dependent.
  - As can be seen that it maps to `SkBitmap::kARGB_8888_Config`
  - The corresponding mesa format is `MESA_FORMAT_RGBA8888_REV` on
    little-endian; `MESA_FORMAT_RGBA8888` on big-endian.
  - Because it is pre-multiplied, it should use
    `glBlendFunc(GL_ONE, GL_ONE_MINUS_SRC_ALPHA)`
- Android `PIXEL_FORMAT_RGBA_565` is always as stored in a `uint16_t`
  - its in memory layout depends on the host endian
  - The corresponding mesa format is `MESA_FORMAT_RGB565`
- `intel_create_renderbuffer` creates renderbuffers with the formats depend on
  endian-ness.
  - that is, they use mesa formats.
  - because `spantmp2.h` uses GL formats, mesa formats must be mapped to gl
    formats before including `intel_spantmp.h`.

## Buffer and Config

- Contexts and drawables are created with a fbconfig.
- How does fbconfig affect them, especially on their buffers?
- For drawables,
  - a drawable is a fbo
  - whether double buffers determines the default render buffer
  - the red/green/blue/alpha size determines the formats of the render buffers
  - the depth/stencil/etc size determines how many render buffers are allocated
- For contexts,
  - the fbconfig determines the hw state
  - if context and drawable mismatch, the hw might draw out of the bounds of the buffer
- For the native Window/Pixmap,
  - because the backing buffer of the front left buffer is the Window/Pixmap itself,
    if Window/Pixmap and the fbconfig of the drawable mismatch, weird color, skewed
    images are expected.
  - actually, the Window must be created with the visual by calling
    `glXGetVisualFromFBConfig` on the fbconfig.
- When doing texture-from-pixmap,
  - one must be able to derive a fbconfig from the pixmap.  A possible route is
    `pixmap -> corresponding window -> corresponding visual -> corresponding fbconfig`
    - For egl, it may call `eglChooseConfig` with `EGL_MATCH_NATIVE_PIXMAP = pixmap`.
  - `glBindTexImage` must use the fbconfig of the pixmap to determine the
    `format` and `type` of the data.  The `internalFormat` may preferable be chosen
    so that zero-copy is possible.
- In X, there are three resources `GLXWindow/Window/DRIDrawable`
  - `Window` is X window
  - `GLXWindow` is Window plus dri driver drawable
  - `DRIDrawable` is merely a buffer

## `GL_KHR_blend_equation_advanced`

- gallium support
  - when `PIPE_CAP_FBFETCH` and/or `PIPE_CAP_FBFETCH_COHERENT` are supported,
    it is emulated on top of `GL_EXT_shader_framebuffer_fetch`
  - there is `PIPE_CAP_BLEND_EQUATION_ADVANCED` for native hw support, but
    only virgl uses it
- `lower_blend_equation_advanced` is always called from `link_shader`, to do
  the emulation
- `glBlendBarrier` (and `glFramebufferFetchBarrierEXT`) calls
   `pipe_context::texture_barrier(PIPE_TEXTURE_BARRIER_FRAMEBUFFER)`
  - on radeonsi, it sets a few cache flush bits.  At draw time,
    `sctx->emit_cache_flush` emits the flush cmds before the draw cmds

## ASTC

- `st_create_context_priv`
  - `transcode_astc` is by default false
    - when true, astc is transcoded to dxt5
  - `has_astc_2d_ldr` means `PIPE_FORMAT_ASTC_4x4_SRGB` support
  - `has_astc_5x5_ldr` means `PIPE_FORMAT_ASTC_5x5_SRGB` support
  - if transcoding, `st_init_texcompress_compute` is called
- `st_init_texcompress_compute`
  - `bc1_endpoint_buf` is a `pipe_buffer` containing dxt tables
  - `astc_luts` are `pipe_sampler_view`s for astc luts
  - `astc_partition_tables` is a hash table of `pipe_sampler_view`s
- `st_destroy_texcompress_compute` destroys all the resources
- texture upload
  - when `st_compressed_format_fallback` returns true, `st_MapTextureImage`
    returns a pointer to a temporary cpu buffer
  - `st_UnmapTextureImage` maps the `pipe_resource` and "copies" the texture
    data into the pipe resource
    - it can decompress directly into the pipe resource using cpu
    - it can decompress to a temporary buffer, compress to a different
      format, copies the transcoded data into the pipe resource using cpu
    - it can call `st_compute_transcode_astc_to_dxt5` to transcode from astc
      to dxt5 using gpu
- `st_compute_transcode_astc_to_dxt5`
  - `cs_decode_astc` decodes astc to rgba8
    - the shader source is in `astc_decoder.glsl`
    - a sampler view is created on demand, holding the partition table of the
      given block size
    - another sampler view is created on demand, holding the astc data
    - a compute job is dispatched
  - `cs_encode_bc3` encodes rgba8 to bc3
