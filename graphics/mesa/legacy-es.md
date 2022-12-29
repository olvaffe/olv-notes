Mesa and OpenGL ES
==================

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

## `_tnl_draw_prims`

- Now, turn our attension to `_tnl_draw_prims`.
- It wraps the client arrays in `struct vertex_buffer`, which expects the type
  to be float.  Conversion is done if needed, as can be seen from
  `_tnl_import_array`.
- The value of certain attributes (color, normal, and depth) are normalized.
  That is, when the normal has an ubyte value 128, it is normalized to ~0.5f.
- It is clear from the function that `GL_FIXED` is not supported.

## Feature

- A feature may consist of
  - states (`ctx->Blah`, always exist)
  - dispatch (`disp->Blah`, always exist, from XML)
  - vtxfmt (`GLvertexformat`, always exist)
  - driver functions (`dd_function_table`, always exist?)
  - `ctx->Extensions.XXX_xxx` (always exist?)
  - `default_extensions` in `extensions.c` (always exist?)
  - and more
- There are 2 things here
  - define feature (mesa core, .c and .h)
    - all the above
  - implement feature (driver, .c and .h)
    - missing right now

## Slim Body

- Cut off Vertex Specification
  - Keep Current Values (color, normal, multi texcoord, vertex attrib)
  - material?
  - When draw arrays in called, `recalculate_input_bindings` is called.  All
    attributes must have an array.  For those not enabled through
    `glEnableClientState`, `currval` arrays are used.
    - `currval` arrays have the same struct as client arrays do.  They point to
      `ctx->Current`.
    - current attribs are reflected when `vbo_exec_copy_to_current` is called at
      flush time.  It copies the buffered attribs to `ctx->Current` and updates
      the `Size` info of the `currval` arrays.
  - due to above, it cannot disable parts of vbo and use api_noop.
    `vbo_exec_FlushVertices_internal` must be called at suitable places.
