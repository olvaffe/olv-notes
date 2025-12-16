Gallium Auxiliary
=================

## Threaded Context

- pipe drivers can call `threaded_context_create` to create a
  `threaded_context` to wrap their `pipe_context`
  - it is nop if `util_get_cpu_caps()->nr_cpus <= 1`
    - `GALLIUM_THREAD` can override
  - `util_queue_init` creates a `gdrv` thread to process jobs
- `threaded_context` handles different context functions differently
  - some functions are passed through
    - e.g., `tc_create_sampler_view` calls `pipe->create_sampler_view`
      directly
  - some functions are batched
    - e.g., `tc_set_sampler_views` calls `tc_add_slot_based_call` to record
      the call into the current batch
  - some functions require synchronization
    - e.g., `tc_texture_map` calls `tc_sync_msg` to wait the current batch
  - the pipe context fluch function is `tc_flush`
    - if the flush is async or can be deferred, it calls `tc_add_call` to
      record the call to the current batch and calls `tc_batch_flush` to flush
      the current batch
    - otherwise, it calls `_tc_sync` to wait for the `gdrv` thread to become
      idle and calls `pipe->flush`
- `tc_batch_flush` calls `util_queue_add_job` to submit the current batch
  - `gdrv` thread calls `tc_batch_execute` to execute the batch
  - `tc->execute_func` is the dispatch table
  - `set_sampler_views` is dispatched to `tc_call_set_sampler_views`
    - it calls `pipe->set_sampler_views`
  - `flush`, when async or deferred, is dispatched to `tc_call_flush` or
    `tc_call_flush_deferred`
    - they call `pipe->flush`
- `_tc_sync` waits for `gdrv` to become idle and also calls `tc_batch_execute`
  to execute the current batch directly

## TGSI: fragment

- The machine works on a quad at a time
  - `mach->QuadPos` gives the coordinates of the quad
  - `mach->InterpCoefs` gives the parameters of the interpolation
  - `mach->Consts` gives the constants (uniforms, etc.)
- before the machine runs, declarations for inputs are executed
  - they specify the inputs and how interpolation is done (constant, linear,
    or perspective)
  - `mach->Inputs` are interpolated according to the quad pos, interpolation
    params, and the declarations.
- after the machine runs, the outputs are stored according to output decls
  - output i is color; output j is position; etc.
  - these info are retrieved by `tgsi_scan_shader`.
- Actually, before the machine runs,
  - input declarations are examined and the draw module is taught how to emit
    vertices. `softpipe_get_vertex_info`.
  - the emitted vertices are used by `setup_context` and `quad_header` are set
    up with correct interpolation coefficients.
  - `quad_header` are then used to prepare the machine

## TGSI: vertex

- Generally in `draw_pt_arrays`, nothing is forced or bypassed.  This means 
  fetch-shade-pipeline get hit a lot, with or without `PT_PIPELINE`.
  - shading happens before pipeline.  pipeline only gets to see the outputs of
    the shader.
  - post-vs clips, does perspective-division and viewport transform.
    - remember vs replaces only model-view and projection.
    - see `post_vs_cliptest_viewport_gl`.
- Unlike fs, `mach->Inputs` are prepared by the caller, as can be seen in
  `vs_exec_run_linear`.

## Draw: Overview

- a pipe driver can use draw module for all operations up until rasterize.
- a pipe driver must install the rasterize stage through
  `draw_set_rasterize_stage`.
- when run, the draw module will run the primitives through the stages
- usually, a driver installs vbuf-based rasterize stage.  See `vbuf_render`.

## Draw: The draw module

- Functions of the draw pipeline
  - vertex fetch
    - non-indexed or indexed with different index sizes
    - vertices on multiple vertex buffers with different formats
    - fetching vertices to `vertex_header` is slow and heavy
  - vertex shader and geometry shader
    - the vertices should have been normalized to `vertex_header`s before both
      shaders
    - the primitive should have been assembled before geometry shader
  - stream output
    - easy
  - coordinate transformation
    - easy
  - pipeline or emit
    - emit is straightforward, but emitting a vertex may be slow
    - pipeline is needed if clipping or other steps are needed.  Or if the pipe
      driver does not support the primitive type.
- Limitations of the draw pipeline
  - the pipe driver has a limit on the buffer size.  Ergo,
    `draw_pt_emit_prepare` has a limit on the number of vertices in a single
    emit.
    - This is usually smaller than the other limitations below.  It can be
      worked around by making emit smart.
  - Another limitation is impl limitation: `UNDEFINED_VERTEX_ID` (65535).  Every
    `vertex_header` is assigned an id when it is emitted in vbuf stage.  It is
    so that we know the vertex has been emitted and the id is the index of the
    vertex in the vertex buffer.
    - This limitation does not exist!  vbuf always `check_space` and it sets
    `max_vertices` below `UNDEFINED_VERTEX_ID`.
  - Another impl limitation is  `DRAW_PIPE_MAX_VERTICES` (4096).  Complex
    primitives are decomposed into simpler ones.  The simpler ones need
    additional flags associated with the vertices such as
    `DRAW_PIPE_RESET_STIPPLE` or `DRAW_PIPE_EDGE_FLAG_0`.  Middle ends use
    `ushort` for vertex indices.  With the upper bits used for flags, the
    available space for real indices is limited.
    - This is real limitation.  But what if we don't decompose in vcache?  emit
      must lift its `max_vertices` limitation (which is ok).  The GS must handle
      more types (which is also ok).  The pipe driver must handle more types
      (which is ok too).  So, why do vcache and varray decompose in the first
      place?  For flatshading on first vertex?
- When do assemble and decompose a primitive?
  - assembly must happen before geometry shader
  - decompose must happen before pipeline (if used)
  - decompose is also needed so that we obey the max vertices limitation
    - decompose a really big triangle strip to triangles
  - decompose must also set vertex flags (reset stipple, ...)
  - geometry shader might add vertices
- What if we decompose only before pipeline, not in vcache or varray?
- Some guidelines
  - A run that uses N vertices might use only M distinct vertices.
    - `N == M` if non-indexed
    - `N >= M` if indexed
    - Always fetch and emit M vertices because they are slow operations
      - vcache fails this because there is no `vertex_id` yet

## Draw: `draw_pt`

- The main entry point of the draw module: `draw_arrays`
- There two frontends
  - `draw_pt_vcache` for indexed drawing or when pipeline is needed
  - `draw_pt_varray` for non-indexed non-pipeline drawing
- Unless vbuf-based rasterize stage is available, all renderings will go through
  the pipeline
- There are 4 middle-ends
  - `draw_pt_fetch_emit` when no shading or pipeline
  - `draw_pt_fetch_shade_emit` when no pipeline
  - `draw_pt_fetch_shade_pipeline` otherwise
  - Alternatively, `draw_pt_fetch_shade_pipeline_llvm` replaces the above when
    LLVM is available
- Middle-end has
  - `run` for the general fetch and draw
  - `run_linear` for non-indexed fetch and non-indexed draw (and non-pipeline)
  - `run_linear_elts` for non-indexed fetch and indexed draw (and non-pipeline)

## Draw: Frontends

- `draw_pt_varray`
  - It will simply call `middle->run_linear` mostly.
  - In case of line loop, triangle fan, or polygon, it needs to call
    `middle->run`
- `middle->run` takes `fetch_elts`, `fetch_count`, `draw_elts`, `draw_count`
  - the fetch ones are indices into the vertex buffer
  - the draw ones are the index buffer of the fetched vertices
  - it is more clear in `draw_pt_cache`
- `draw_pt_cache`
  - It tries to call `middle->run_linear_elts` in `vcache_check_run` for the
    fast path.  If failed, it will call `middle->run` in the general
    `vcache_run`.  If the middle end needs pipeline, it will simply call
    `vcache_run_extras`.
  - The idea is that, to run the pipeline, we need additional flags.  We must
    use the slow `vcache_run_extras`.

## Draw: Middle-end

- pipeline middle-end, among others, has two important submodules
  - `draw_pt_fetch` turns vertex attributes into the standard 4-float format for
    consumption by the shader and the pipeline
  - `draw_pt_emit` turns standard 4-float outputs from the pipeline to the
    hardware format
