Mesa and Shader
===============

## Shaders

- vertex shaders are run once for each vertex.  It manipulates attribs of the
  vertex, including position and transforming to on-screen coordinates.
- geometry shaders are run on meshes.  It can add or remove vertices from
  meshes.  It is not supported by mesa.
- rasterizer takes inputs from geometry shader and outputs pixels to fragment
  shader.
- fragment(/pixel) shaders are run on fragments.  It adds effects like lighting
  or blurring.
- shaders are in `ctx->Shader`

## Shader

- shader calls into driver's `CreateProgram`, which calls into
  `_mesa_create_shader`.
- other calls like `NewProgram` or `BindProgram` call into `i915NewProgram` or
  `i915BindProgram`.

## Programs

- in main context,
  - `ctx->FragmentProgram.Current` stores current bound fragment program
  - `ctx->VertexProgram.Current` stores current bound vertex program
  - `ctx->Program` stores the error status of the program parser
- Their `_Current` decides which program to use in `update_program`.
  - It might be `ctx->Shader.CurrentProgram->FragmentProgram`, the shader
  - It might be their `Current`, the program itself
  - It might be from `_mesa_get_fixed_func_fragment_program`, fixed-function
    pipeline written in program.  This is used only when `MESA_TEX_PROG` or
    `MESA_TNL_PROG` env are defined.
  - It might be `NULL`, fixed-function pipeline in old code path.

## Rasterization

- See Fig 3.1 of the spec.
- Vertices are assembled into points, lines, or triangles.
- They are rasterized to produce fragments.
- Besides, `glDrawPixels` and `glBitmap` can also be rasterized to produce
  fragments.
- After rasterization, the results are passed to fixed-function pipeline or
  fragment program to produce the final fragments.
- Per-fragment operations do a number of tests and blendings on the fragments
  produced in the rasterization stage before they reach the framebuffer.
