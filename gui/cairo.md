Cairo
=====

## `cairo-gl`

- Basic Usage
  - `dev = cairo_egl_device_create(egl_dpy, egl_ctx);`
  - `surf = cairo_gl_surface_create(dev, CAIRO_CONTENT_COLOR, 300, 300);`
    creates a texture-based surface.
  - `surf = cairo_gl_surface_create_for_texture(dev, CAIRO_CONTENT_COLOR, tex, 300, 300);`
    creates a texture-based surface from an existing `GL_TEXTURE_2D` (unless
    `GL_ARB_texture_non_power_of_two` is not suppported) texture.
  - `surf = cairo_gl_surface_create_for_egl(dev, egl_surf, 300, 300);` creates a
    `EGLSurface`-based surface from an existing `EGLSurface`.
- device creation
  - after creation, the `EGLContext` belongs to the device.  It will be made
    current and not current by cairo.
  - before rendering to a surface, `_cairo_gl_context_set_destination` is
    called.  A device (i.e. a gl context) can render to a surface at a time.  It
    is called the current target.
    - for a texture-based surface, an FBO is created and bound.  The texture is
      attached to the fbo
    - for an `EGLSurface`-based surface, the surface is made current.
