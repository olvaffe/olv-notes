Gallium softpipe
================

## `egl_softpipe.so`

- See also `mesa-egl`, same section.
- From a very high level, programmers calls OpenGL API.  The API is implemented
  by st.  st implements it by calling pipe driver.  For stuff like buffer
  allocation, pipe driver calls pipe winsys, without st knowing it.
  - From EGL's view, EGL is pipe winsys.  It calls pipe driver and st to bind
    them together.
- `softpipe_create_screen`
  - A subclass of `struct pipe_screen` is returned.
  - It calls `u_simple_screen_init` to pass calls to pipe screen methods
    directly to pipe winsys methods.
- `softpipe_create`
  - A subclass of `struct pipe_context` is returned.
  - Every EGL context is backed by a pipe context and a st context, backed still
    by that pipe context.

## Quad

- In `setup_tri`, triangle is rasterized.
  - The 3 vertices are ordered by their y, and are remembered in `vmin`, `vmid`,
    and `vmax`.
  - `ebot` describes the edge from `vmin` to `vmid`.
  - `etop` describes the edge from `vmid` to `vtop`.
  - `emaj` describes the edge from `vmin` to `vtop`.
