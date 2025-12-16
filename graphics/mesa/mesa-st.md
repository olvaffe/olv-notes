Mesa and Gallium
================

## Idea

- gallium expects the hw to be able to manage buffer objects and support vertex
  and fragment shaders.
- In `st_create_context_priv`, `ctx->FragmentProgram._MaintainTexEnvProgram` is
  set to true to use a program to replace fixed-function.
  - In `_mesa_update_state_locked`, many states cause the program to be
    regenerated.
  - When a texture unit is used, the corresponding flag in
    `ctx->FragmentProgram->_Current->Base.SamplersUsed` is set.
  - A sampler is a texture unit.

## States

- The pipe context is asked to create, bind, and delete various states.
  - Binding a state usually marks it for lazy binding.
  - st usually uses cso to cache the created states.  It is also used to make
    sure the state has really changed before asking pipe context to bind.
- Sampler state
  - When `glTexEnv` or `glTexParameter` is called, mesa marks `_NEW_TEXTURE`.
    This causes st to create and bind a new sampler state when updating states.
    A new sampler state template is passed to the pipe context.  The template is
    remembered by the pipe context and the contents are translated to register
    values.  Some of the values are not translated.  They might be used by, say,
    pipeline.
  - Incidentally, this state decides which samplers are used.  The decision is
    made by looking at the vertex and fragment programs, which are regenerated
    in `update_program` to reflect the real info.
- Blend state
  - When blending is enabled or disabled, or blending function is changed, mesa
    marks `_NEW_COLOR`.  This causes st to create and bind a new blend state.
    Again, in the pipe context, the blend template is translated to register
    values.
- There are more states such as depth, rasterizer, fs, vs, etc.
- There are also parameter-like states
  - These states are set, intead of create, bind, and delete.
  - They include blend color, clip state, constant buffer, viewport, scissor,
    framebuffer state, vertex buffer, sampler textures, and etc.
  - Especially, the color buffers and the depth buffer are passed through
    framebuffer state.  The vertex buffers are through vertex buffer state.  The
    textures are through sampler textures state.  Constant buffer is through
    constant buffer state, but for i915simple, it is copied out to system memory.
- `set_viewport_state`
  - all vertices are scaled and translated with the viewport after clipping
  - it is set in such a way that, when the target is a window, `[-1, -1]` is
    transformed to `[x, fb_height - y]` and `[1, 1]` is transformed
    `[x + w, fb_height - (y + h)]`, where `x, y, w, h` is given by `glViewport`
- When `pipe_sampler_state::normalized_coords` is true, the texture coordinates
  are scaled by the width, height, and depth of the texture before addressing
  the texels.

## st Buffer Management

- `st_cb_bufferobjects.c`
  - The storage of an st buffer object is a pipe buffer allocated by
    `pipe_buffer_create`.
- `st_cb_texture.c`
  - An st texture object
    - is created when `glGenTextures`.
    - has a pipe texture as its storage, when `glTexImageXD` or the likes is
      called.  This pipe texture is allocated by `st_texture_create`.
      It is allocated by the pipe screen and the storage is normally a pipe
      buffer. 
    - has many st texture images.  The storage of an image comes from the
      object's pipe texture or malloc.  The data might be client buffer or pack
      from a PBO.
    - In `st_set_teximage`, it is possible to set a pipe texture to be an st
      texture image's storage.
- `st_cb_fbo.c`
  - An st renderbuffer is created with `st_renderbuffer_alloc_storage` as its
    `AllocStorage`.  The storage might be a malloc memory stored in `Data`, or
    a pipe surface which is created from an image of a pipe texture, which is
    created by pipe screen's `texture_create`.
  - Or, as can be seen in `st_render_texture`, an st renderbuffer might be based
    on an st texture object.  This is for render-to-texture.
  - Renderbuffers are attached to FBO.
- Summary
  - The storage is always pipe buffer.  A pipe texture is based on pipe buffer
    and has many levels.  It is not easy to operate on a pipe texture, so a pipe
    surface is used to wrap a pipe texture level for manipulating.
  - Renderbuffer is based on pipe surface.  Because a pipe surface wraps a pipe
    texture, render-to-texture becomes natural.

## Atom

- Atom is used to register a lazy state update.

## `glTexImage2D`

- core `_mesa_TexImage2D`
  - `_mesa_lock_texture`
  - call `ctx->Driver.FreeTexImageData` if there are data
  - `clear_teximage_fields` and `_mesa_init_teximage_fields`
  - `_mesa_choose_texture_format`
    - different levels may have different formats
  - `ctx->Driver.TexImage2D`
  - `_mesa_set_fetch_functions`
  - `check_gen_mipmap`
  - `_mesa_unlock_texture`
- `st_FreeTextureImageData`
  - unreference the image pt if exists
  - free the image buffer if exists
- `st_TexImage2D`
  - free the old buffer of the image
  - free `stObj->pt` if it is in any way incompatible
  - allocate `stObj->pt` by guessing if none exists
  - set `stImage->pt` to `stObj->pt` if they are compatible
  - upload the pixel data directly into `stImage->pt` if possible; otherwise,
    upload them into a temporary buffer owned by the image
- core `update_texture_state`
  - `_mesa_test_texobj_completeness`
- `st_finalize_textures`
  - the goal is to finalize `stObj->pt`
  - `stObj->pt` allocation
    - if `firstImage->pt` suffices, use it
    - otherwise, allocate one with the same format as that of the first image
  - `stObj->pt` contents
    - the contents of each image are uploaded into `stObj->pt`
    - the buffers of the images are discarded.  Each `stImage->pt` will point to
      `stObj->pt`
- `update_textures`
  - create a sampler view from `stObj->pt`.  A sampler view may have a different
    but compatible pipe format and a swizzle
  - `stObj->base.DepthMode` specifies how to interpret a depth texture as a
    color texture
  - `stObj->base._Swizzle` specifies the swizzle of the texture
  - call `set_fragment_sampler_views` and `set_vertex_sampler_views`
- `update_samplers`
  - update the samplers' states
  - call `bind_fragment_sampler_states` and `bind_vertex_sampler_states`

## Texture Buffer Management

- The pipe texture of an st texture object is allocated by
  `guess_and_alloc_texture`
  - The pipe texture created can hold a whole mipmap set, unless it is guessed
    not needed.
- An st texture image re-uses the pipe texture of its st texture object if it is
  tested to match by `st_texture_match_image`.
- Usually, the data of the images are hold in the pipe texture.  However, it is
  possible that some images hold data in its own buffer or use a pipe texture
  from other sources (texture from pixmap).
- When mesa wants to flush vertices, it calls `st_draw_vbo`, which calls into
  pipe context's `draw_arrays` for each primitives (a `glBegin/glEnd` gives a
  primitive).  Take `softpipe_draw_arrays` for example,
  - It enters the `draw` modules and goes through frontend, middleend and
    backend.  It begins from `draw_pt_arrays`
  - The backstrace is something like

    varray_run
    varray_flush_linear
    fetch_pipeline_linear_run
    draw_pt_emit_linear
    sp_vbuf_draw_arrays
    setup_tri
    subtriangle
    flush_spans
    emit_quad
    earlyz_quad
    depth_test_quad
    shade_quad
    exec_run
    tgsi_exec_machine_run
    exec_instruction
    exec_tex
  - In `fetch_pipeline_linear_run`, the middle layer
    - vertex inputs are fetched and translated into a format for vertex shader
    - The vertex shader is run
    - post vs is run for clipset, rhw divide, viewport transform.
    - `draw_pt_emit_linear` is called and the vertices are translated again.
    - They are translated into a output buffer prepared by backend
  - In `sp_vbuf_draw_arrays`, the `vbuf_render` backend
    - The final vertices (with screen coordinates) are parsed and the result is
      stored in `setup_context`.
    - It is converted into spans and sent to softpipe in quads.  That is, 2x2
      blocks.  See `softpipe->quad`.
  - When the pipe driver does not use vbuf, it installs its own rasterization
    stage.  If it uses vbuf, it provides `struct vbuf_render` and installs
    `draw_vbuf_stage`.
    - vbuf has the advantage of skipping pipeline.  It can be enabled by
      `draw_set_render`.  When enabled, vertices are emitted directly to the
      pipe driver when pipeline is not needed.
  - It enters `exec_tex` of tgsi for texture lookup.
  - `get_samples` of the sampler points to `sp_get_samples_fragment`.  It
    returns `QUAD_SIZE` samples, currently 4.  The softpipe implementation uses
    a cached tile to transfer 64x64 (It takes `64 * 64 * 4 * sizeof(float) =
    64KB`) texels at a time.  The transfer uses a pipe transfer.
  - There is a helper function called `pipe_get_tile_rgba`.  It transfers the
    texels and unpacks it from pipe texture format to rgba.

## Vertex Buffer

- In `st_draw_vbo`, vertex buffer is represented by many
  `struct gl_client_array`.
- The pipe context expects `struct pipe_vertex_buffer` and
  `struct pipe_vertex_element`.
  - A pipe vertex buffer is based on a pipe buffer for its storage. `stride`
    gives the size of a vertex and `max_index` gives the number of vertices in
    the buffer.  Usually, each attr of a vertex is described by a pipe vertex
    buffer, even if all attrs reside on the pipe buffer.  This is possible
    because of `buffer_offset`.
  - A pipe vertex element describes the info of an attribute.
- For softpipe, these infos are passed to `draw_context's pt` and `draw_arrays`
  is called.  `pt` stands for pass-through.  `fse` stands for fetch-shade-emit.
- The frontend might be `varray` or `vcache`.  The middle-end might be one of
  the middle-ends initialized in `draw_pt_init`.
- Quote `draw_pt_fetch_emit.c`

    The responsibilities of a middle end are to:
     - perform vertex fetch using
          - draw vertex element/buffer state
          - a list of fetch indices we received as an input
     - run the vertex shader
     - cliptest, 
     - clip coord calculation 
     - viewport transformation
     - if necessary, run the primitive pipeline, passing it:
          - a linear array of vertex_header vertices constructed here
          - a set of draw indices we received as an input
     - otherwise, drive the hw backend,
          - allocate space for hardware format vertices
          - translate the vertex-shader output vertices to hw format
          - calling the backend draw functions.
- The middle end fecthes the vertices, process them, and emit them.
  - fetch can be done with the help of `draw_pt_fetch.c`.  Each fetched vertex
    will have `struct vertex_header` prepended.  The fetched format will be
    `PIPE_FORMAT_R32G32B32A32_FLOAT`.
  - vs is run in-replace on the buffer of the fetched vertices if `PT_SHADE` is
    set.
  - post-vs is then run.
  - It then runs the `struct draw_stage` pipeline or emits directly
  - emit is done with the help of `draw_pt_emit.c`.  The backend is asked for
    `struct vertex_info`.  It has enough info to describe the output formats.
    The emitted data have `struct vertex_header` stripped.  After emitting,
    `render->draw_arrays` is called.
- The pipeline of the middle end
  - is run by `draw_pipeline_run_linear`.
  - is reset by `draw_pipeline_flush`.
  - the vertices are put into `struct prim_header`s and one of the pipeline's
    `point`/`line`/`tri` is called on them, one at a time.
    - e.g., to make stipple line, the stipple stage can emit multiple line
      segments for a single line.
  - The last stage of this pipeline is `rasterize`.  It is set by
    `sp_init_vbuf`.  It does something similiar to `draw_pt_emit.c`.  The
    `struct vertex_header` is also stripped.  It buffers all vertices until the
    buffer is full or flushed.  It calls `render->draw` to draw the vertices.
- The emitted vertices, through direct emission or pipeline, go through
  per-fragment operations (`struct quad_stage`) (Fig. 4.1 of OpenGL 3.1)
  - The vertices describing a point/line/tri is parsed and turned into
    `struct quad_header`s.  per-fragment ops are done on quads, 2x2 blocks.
- On `i915simple`, the last stage of the middle end pipeline
  - is initialized in `i915_draw_vbuf_stage`.
  - The buffered vertices are submitted directly to the hardware
  - That is, per-fragment ops are done in pure hardware.
  - It works because the hw states are updated to reflect new vbo, color buffer,
    depth buffer, textures, etc.  Grep for `OUT_RELOC`.
  - Two operations are also accelerated: `surface_copy` and `surface_fill`.
  - Incidentally, xf86-video-intel uses the same functions for UXA copy and
    fill.  For composite, it uses FS.

## Shaders

- Mesa FS or VS is translated into `struct pipe_shader_state`, which is a list
  of TGSI tokens.
- The tokens is passed to pipe driver's `create_fs_state` or `create_vs_state`
  to create a driver specific state.
  - In the case of softpipe, `struct sp_fragment_shader` and
    `struct sp_vertex_shader` are created.
- The state is used when one of `bind_fs_state` or `bind_vs_state` is called.
  - It binds into softpipe context's `fs` and `vs` directly.

## `glDrawPixels`

- It is implemented in `st_DrawPixels`.
- `st_make_passthrough_vertex_shader` is called to create passthrough vp
  - It does only two things.
  - `gl_Position = gl_Vertex;`
  - `gl_TexCoord[0]  = gl_MultiTexCoord0;`
- A fp that combines pixel-transfer-fp and current-fp is created.
- A temp pipe texture is created and the pixels are transferred into it.
- Finally, `draw_textured_quad` is called
  - It saves current states and installs its own states
  - It uses 4 vertices to describe the rectangle.
  - Normally vertex buffer path is taken to draw.
- Similiar pathes are taken for `glBitmap` and `glClear`.

## Shader Translation

- `find_translated_vp` translates mesa programs to TGSI.
- Have a look at mesa IO
  - vs inputs are `VERT_ATTRIB_POS`, etc. (total ~32).
  - vs outputs are `VERT_RESULT_HPOS`, varying, etc. (total ~32).
  - fs inputs are `FRAG_ATTRIB_WPOS`, varying, etc. (total ~28)
  - fs outputs are `FRAG_RESULT_DEPTH`, etc. (total ~6)
- fs inputs are parsed and `stfp->input_to_slot` stores how to map frag input to
  TGSI slot.  When translating,
  - 

## Flush

- There are many places that might buffer.
- The draw module pipeline
  - vbuf might buffer the vertices.  See `vbuf_flush_vertices`.
  - `draw_flush` to flush the pipeline.
  - Flush should happen whenever any of the state sets changed.
- The softpipe driver pipeline
  - The fragment output might be on the cache tile.  See `output_quad`.
  - `sp_flush_tile_cache` flushes the caches.
  - Texture samples might be cached.  See `sp_get_cached_tile_tex`.
  - `sp_flush_tile_cache` or setting `modified` flag can be used to flush the
    cache.
- `st_flush`
  - mesa might buffer vertices.  `FLUSH_CURRENT` to flush.
  - `glBitmap` might be buffered in st.  See `st_flush_bitmap`.
  - `glClear` is implemented with a weird slot idea.  See `st_flush_clear`.
  - `BlitFramebuffer` also uses weird slot.  See `util_blit_flush`.
  - Same to mipmap generation... See `util_gen_mipmap_flush`.
- Flush flushes commands to hw, and returns a fence.  The fence can be used to
  wait the flushed commands to complete.  It is used to implement `glFinish`.

## GL pipeline and Gallium

- Vertices are processed by vertex shader.
  - The fixed-function vertex shader does model-view and projection
    transformations, and ...
  - texture coords and normals transformations
  - per-vertex lighting
  - and more
- The outputs of a vertex shader are assembled into primitives.  They are sent
  to run pipeline.
- An optional first stage of the pipeline is the geometry shader.
  - There is no fixed-function equivalent for geometry shader.
- The rest of the pipeline does
  - the coordinates are normalized (perspective division).
  - viewport mapping is performed.
  - the above two stages are called post vs in gallium.
  - flatshading
  - clipping.  New vertices might be added and interpolated.
  - final color processing
- Primitives are then sent for rasterization.
  - In OpenGL 2.x, besides primitives, there are two more sources that need
    rasterization.  They are `glDrawPixels` and `glBitmap`.
- The outputs of rasterization are fragments.  They are sent to a fragment
  shader for further processing.
  - The fixed-function fragment shader does texturing, color sum, fog, ...
  - and more
- The resulting fragments go to per-fragment operations.
  - including scissor test, depth test, blending, etc.

## `winsys/drm/`

- A state tracker has access to `pipe_screen` and `pipe_context`
  - they are abstraction of the hardware
  - they are provided by pipe drivers
  - `pipe_context` represents the hw pipeline
  - `pipe_screen` represents everything except pipeline
    - buffer/user buffer/surface buffer
      - user buffer is a buffer with user-specified data
      - surface buffer is usually a buffer the special alignment
    - texture
      - texture is a buffer with associated information for use as a texture
    - ...
  - a `pipe_context` is created with a `pipe_screen` to represent the hw as a
    whole
- `winsys/drm/`
  - for integration with the window system, `pipe_screen` is created with a
    driver-specific winsys
  - this is implementation details
  - the winsys might inherit `pipe_winsys`, or might not
  - winsys is in charge of the creation of buffer/user buffer/surface buffer
    - `pipe_screen` simply calls to its winsys
  - In `winsys/drm/`, X11 is _not_ the winsys.  `/dev/dri/card0` is the
    winsys.  This is not intuitive.
  - A `pipe_screen` might be used with GL state tracker.  But it might also be
    used with xorg state tracker to implement a DDX driver.  If a winsys abstracts
    X11, the latter application is not possible
- `struct drm_api`
  - is capable of creating `pipe_screen` and `pipe_context`
  - `/dev/dri/card0` is used as the underlying winsys
  - besides, it provides callbacks to
    - create `pipe_texture` from GEM names
    - return the GEM name of a `pipe_texture`
    - return the GEM handle of a `pipe_texture`
  - this is why it can be used with DRI2 protocol for X11 integration
- `struct x11_api`
  - similar to `struct drm_api`, but at a higher level
  - `create_screen` and `create_context` is the same
  - new stuff! `create_framebuffer`!

## Buffer Objects

- Target to gallium binding mappings
  - `GL_PIXEL_PACK_BUFFER` or `GL_PIXEL_UNPACK_BUFFER`
    - `PIPE_BIND_RENDER_TARGET | PIPE_BIND_SAMPLER_VIEW`
  - `GL_ARRAY_BUFFER`
    - `PIPE_BIND_VERTEX_BUFFER`
  - `GL_ELEMENT_ARRAY_BUFFER`
    - `PIPE_BIND_INDEX_BUFFER`
- Usage mappings
  - `GL_STATIC_DRAW`, `GL_STATIC_READ`, or `GL_STATIC_COPY`
    - `PIPE_USAGE_STATIC`
  - `GL_DYNAMIC_DRAW`, `GL_DYNAMIC_READ`, or `GL_DYNAMIC_COPY`
    - `PIPE_USAGE_DYNAMIC`
  - `GL_STREAM_DRAW`, `GL_STREAM_READ`, or `GL_STREAM_COPY`
    - `PIPE_USAGE_STREAM`
- How pipe drivers handle them?
  - for vertex/index buffer, use GTT
  - otherwise, for dynamic/stream, use GTT
  - otherwise, for static, use VRAM

## Samplers and Sampler Views

- the dirty bits are `ST_NEW_SAMPLERS` and `ST_NEW_SAMPLER_VIEWS`
  - they are set in `st_invalidate_state` when some prior GL call set
    `_NEW_TEXTURE_OBJECT`
  - also somewhat indirectly when some prior GL call set `_NEW_PROGRAM`
    - when a program is linked, `st_link_nir` calls
      `st_set_prog_affected_state_flags` to decide states that the program
      invalidates
    - upon program change, `st->gfx_shaders_may_be_dirty` is set
    - `st_validate_state` calls `check_program_state` to mark the affected
      states dirty
- `st_validate_state` calls "atoms" depending on the dirty states
- `st_update_fragment_samplers`
  - when a program is linked, `gl_nir_link_uniforms` sets `prog->SamplersUsed`
    and `prog->SamplerUnits`
    - every glsl uniform that is a sampler is assigned an sequential index
    - `prog->SamplersUsed` is a mask of all samplers
    - `prog->SamplerUnits` maps a glsl sampler to a gl texture unit
      - it is updated by `glUniform1i`
      - or using `layout(binding=X)` in glsl
  - `st_update_fragment_samplers` updates `st->state.frag_samplers` based on
    `SamplersUsed` and `SamplerUnit`
    - IOW, `pipe_context::bind_sampler_states` indices are the
      driver-generated sequential indices of glsl sampler uniforms
- `st_update_fragment_textures`
  - `pipe_context::set_sampler_views` indices are the driver-generated
    sequential indices of glsl sampler uniforms

## `st_draw_vbo`

- When vertices are flushed, `st_draw_vbo` (gallium) or `_tnl_draw_prims` is
  called.  We want to have a closer look at `st_draw_vbo` here.
- First, we look at the memebers of a client array
  - `Ptr` points to the value of the attr of the first vertex.  It may be
    userspace buffer or offsets to the buffer object.
  - `Type` is the type (float, fixed, etc.) of the value.
  - `Size` is the size of the attr (1, 2, 3, or 4).
  - `Stride` is the stride given by user.
  - `StrideB` is the stride to the next vertex.  For user supplied arrays, if
    `Stride` is not given, it is calculated from `Type` and `Size`.
- A client array might be from the user, built from vbo, or wraps
  `ctx->Current.Attrib`.  In the case a `glBegin/glEnd` pair does not contain
  any `glColor`, the client array for color wraps the current attrib.  It has
  `Stride` and `StrideB` equal to zero.  Therefore, no matter which vertex is
  asked, it always dereferences the same memory.
- `st_draw_vbo` wraps the client arrays in `struct pipe_vertex_buffer` and
  `struct pipe_vertex_element`.  They are then passed to pipe context to draw
  the elements.
- The types the wrapping supports can be seen from `st_pipe_vertex_format`.
