EGL
===

## History

- 1.0 was released in 2003
  - based on GLX 1.3
  - OpenGL ES 1.0
- 1.1 was released in 2004
  - OpenGL ES 1.1
  - power management and swap control from IMG
  - render-to-texture based on `WGL_ARB_render_texture`
- 1.2 was released in 2005
  - added OpenVG 1.0 support
  - added OpenGL ES 2.0 support
- 1.3 was released in 2007
  - separate OpenGL ES 1.0 and 2.0 contexts in `eglCreateContext`
  - rename tokens, e.g. `NativeWindowType` renamed to `EGLNativeWindowType`.
- 1.4 was released in 2008
  - added OpenGL support
  - `EGL_SWAP_BEHAVIOR`
- 1.5 was released in 2014
  - `EGL_EXT_client_extensions`
  - `EGL_EXT_platform_base`
  - `EGL_KHR_fence_sync`
  - `EGL_KHR_image_base`
  - `EGL_KHR_create_context`
  - `EGL_KHR_get_all_proc_addresses`
  - `EGL_KHR_surfaceless_context`

## EGL

- `eglSwapBuffers` posts color buffer to a native window.  It has no effect on
  pbuffer or pixmap.  Its effect happens at a finite time later (e.g., next
  vsync).
- Pixmap surface always draws to the native pixmap buffer.  Pbuffer surface
  always draws to the back buffer owned by EGL.  Window surface draw to the back
  buffer owned by EGL, or to the native buffer if the client API supports that.
  OpenGL ES does not support drawing to window's native buffer.
- EGL allows programmer to specify render buffer when creating window surface.
  When `EGL_RENDER_BUFFER` is `EGL_SINGLE_BUFFER`, it renders directly to the
  native window.  When it is `EGL_BACK_BUFFER`, int renders to the back buffer.
  The former is not supported by OpenGL ES.
- EGL defines `eglBindTexImage` and `egReleaseTexImage` to allow programmers to
  use a pbuffer surface as a texture.  The surface must have either
  `EGL_BIND_TO_TEXTURE_RGB` or `EGL_BIND_TO_TEXTURE_RGBA`, as requested when
  creating.  Attribute `EGL_TEXTURE_TARGET` of the pbuffer surface gives the
  texture target.
  - `glCopyTexImage2D` copies current surface.  It involves copy and is limited
    to only current surface.
  - `glXBindTexImageEXT`, the famous `EXT_texture_from_pixmap`

## Double Buffering

- a context may or may not support right buffer and/or back buffer.
  - it is determined at context initialization time out of the OpenGL scope
- front buffer is potentially visible on monitor, while back buffer is
  potentially not.
- the default draw buffer is back buffer if the context is double-buffered.
  otherwise, the default draw buffer is front buffer.
- the same applies to opengl es.  the only difference is that there is no
  `glDrawBuffer`.
- the contents of the buffers are undefined after GL context is initialized.
  That is why `glClear` should be called before drawing.
  - swapping buffers unsing winsys api may make the contents undefined too.
- Theoretically, on X11
  - man DBE
  - GLX works with DBE nicely.  Swapping buffers in either extension is
    reflected in the other extension.
  - A normal X visual is single-buffered.  The corresponding GL context is
    single-buffered too.  The front buffer is the buffer of the X window itself.
    `glXSwapBuffers` is no-op for single-buffered context.
  - A DBE X visual is double-buffered.  The corresponding GL context is
    double-buffered too.  Its front and back buffers correspond to DBE front and
    back buffers.  Usually, OpenGL draws to back buffer and relies on buffer
    swaping for the buffer to become visible.  But it might draw to front buffer
    directly too.  A front/back buffer is still a front/back buffer after
    swapping (see DBE).
- Because `glXSwapBuffers` corresponds to DBE's buffer swapping, if OpenGL
  renders to front buffer directly, a call to `glXSwapBuffers` gives undefined
  contents (because the contents of back buffer are undefined).
- That is, theoretically, OpenGL and GLX work with DBE.  But in reality (because
  of DRI?),
  - A front buffer is still the window's buffer
  - A DRI driver using `driCreateConfigs` usually creates both single-buffered
    and double-buffered configs, regardless of DBE.
  - A back buffer is always allocated by a video driver.
  - swapping buffers copies the contents of driver-allocated back buffer to
    front buffer.

## Color buffers

- OpenGL ES 1.1
  - Chapter 4: The color buffer consists of either or both of a front (single)
    buffer and a back buffer. Typically the contents of the front buffer are
    displayed on a color monitor while the contents of the back buffer are
    invisible. The color buffers must have the same number of bitplanes, although
    a context may not provide both types of buffers.
  - Sectin 4.2: Color values are written into the front buffer for single
    buffered contexts, or into the back buffer for back buffered contexts. The
    type of context is determined when creating a GL context.
    - olv: the last sentence should be "... when associating a framebuffer with
      a GL context".
- OpenGL ES 2.0
  - Chapter 4: For the default window-system provided framebuffer, the color
    buffers consist of either or both of a front (single) buffer and a back
    buffer. Typically the contents of the front buffer are displayed on a color
    monitor while the contents of the back buffer are invisible. The color
    buffers must have the same number of bitplanes, although a context may not
    provide both types of buffers.
  - Section 4.2.1: Color values are written into the front buffer for single
    buffered contexts, or into the back buffer for back buffered contexts. The
    type of context is determined when creating a GL context.
- OpenGL 3.3 compat
  - Section 2.1: Allocation and initialization of GL contexts is also done using
    these companion APIs. GL contexts can typically be associated with different
    default framebuffers, and some context state is determined at the time this
    association is performed.
    It is possible to use a GL context without a default framebuffer, in which
    case a framebuffer object must be used to perform all rendering. This is
    useful for applications needing to perform offscreen rendering.
  - Chapter 4: All color buffers must have the same number of bitplanes,
    although an implementation or context may choose not to provide right
    buffers, back buffers, or auxiliary buffers at all
    - olv: but the front buffer must exist
