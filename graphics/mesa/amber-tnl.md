Mesa and TNL Context
====================

## `draw_prims`

- It is provided by `_tnl_draw_prims`, set by tnl through `vbo_set_draw_func`.
- vbo does not know about TNL.
- TNL only calls several helper functions from vbo and `vbo_set_draw_func`.
- In main context, tnl is `ctx->swtnl_context` and vbo is `ctx->swtnl_im`.
- It collects the inputs into its `vb`, `struct vertex_buffer`.
- It calls driver's `RunPipeline`, which can call back to `_tnl_run_pipeline`.
- `_tnl_run_pipeline` runs pipeline.  Intel driver installs `intel_pipeline`.
  A pipeline is divided into stages
  - vertex transform, cull, normal transform, lightning, fog, texgen, texture
    transform, point attenuation, vertex program, and render
  - all stages are provided by TNL except render.
  - in vertex stage, if `_Current` of `VertexProgram` is non-null, the stage is
    skipped.  The program is executed later in vertex program stage.
- See `intelInitTriFuncs`.
- In render stage, intel
  - `tnl_dd/t_dd_dmatmp.h` is used by `intel_render.c`.
  - `tnl_dd/t_dd_tritmp.h` and `tnl/t_vb_rendertmp.h` are used by
    `intel_tris.c`.
  - `intel_spantmp.h`, `intel_depthtmp.h` `stenciltmp.h` are used by
    `intel_span.c`.
  - uses swrast for output
  - swrast draws renderbuffer using renderbuffer's `PutRow`, etc.
  - These hooks are set up by including `intel_spantmp.h`.

## `_tnl_draw_prims`

- Now, turn our attension to `_tnl_draw_prims`.
- It wraps the client arrays in `struct vertex_buffer`, which expects the type
  to be float.  Conversion is done if needed, as can be seen from
  `_tnl_import_array`.
- The value of certain attributes (color, normal, and depth) are normalized.
  That is, when the normal has an ubyte value 128, it is normalized to ~0.5f.
- It is clear from the function that `GL_FIXED` is not supported.

## Render, the Last Stage

- `struct vertex_buffer` emits its status in the format described by
  `_tnl_install_attrs`.  It is emitted to
  `TNL_CONTEXT(ctx)->clipspace.vertex_buf`.
  - For `swrast`, the format is `SWvertex`.
  - For `i915`, the format is installed in `i915ValidateFragmentProgram` and it
    is `intelVertex` which is defined by `tnl_dd/t_dd_vertex.h`.  The buffer is
    pointed to by `intel->verts`.
