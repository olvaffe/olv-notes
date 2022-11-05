Gallium Draw Module
===================

## Overview

- a pipe driver can use draw module for all operations up until rasterize.
- a pipe driver must install the rasterize stage through
  `draw_set_rasterize_stage`.
- when run, the draw module will run the primitives through the stages
- usually, a driver installs vbuf-based rasterize stage.  See `vbuf_render`.

## The draw module

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

## `draw_pt`

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

## Frontends

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

## Middle-end

- pipeline middle-end, among others, has two important submodules
  - `draw_pt_fetch` turns vertex attributes into the standard 4-float format for
    consumption by the shader and the pipeline
  - `draw_pt_emit` turns standard 4-float outputs from the pipeline to the
    hardware format
