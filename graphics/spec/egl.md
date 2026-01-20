EGL
===

## Specs

- <https://registry.khronos.org/EGL/>

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

## Chapter 1. Overview

## Chapter 2. EGL Operation

- 2.1. Native Platforms and Rendering APIs
- 2.2. Rendering Contexts and Drawing Surfaces
- 2.3. Direct Rendering and Address Spaces
- 2.4. Shared State
- 2.5. EGLImages
- 2.6. Multiple Threads
- 2.7. Power Management
- 2.8. Extensions

## Chapter 3. EGL Functions and Errors

- 3.1. Errors
- 3.2. Initialization
- 3.3. EGL Queries
- 3.4. Configuration Management
- 3.5. Rendering Surfaces
  - EGL allows programmer to specify render buffer when creating window
    surface.  When `EGL_RENDER_BUFFER` is `EGL_SINGLE_BUFFER`, it renders
    directly to the native window.  When it is `EGL_BACK_BUFFER`, int renders
    to the back buffer.  The former is not supported by OpenGL ES.
  - Pixmap surface always draws to the native pixmap buffer.  Pbuffer surface
    always draws to the back buffer owned by EGL.  Window surface draw to the
    back buffer owned by EGL, or to the native buffer if the client API
    supports that.  OpenGL ES does not support drawing to window's native
    buffer.
- 3.6. Rendering to Textures
  - EGL defines `eglBindTexImage` and `egReleaseTexImage` to allow programmers
    to use a pbuffer surface as a texture.  The surface must have either
    `EGL_BIND_TO_TEXTURE_RGB` or `EGL_BIND_TO_TEXTURE_RGBA`, as requested when
    creating.  Attribute `EGL_TEXTURE_TARGET` of the pbuffer surface gives the
    texture target.
    - `glCopyTexImage2D` copies current surface.  It involves copy and is
      limited to only current surface.
    - `glXBindTexImageEXT`, the famous `EXT_texture_from_pixmap`
- 3.7. Rendering Contexts
- 3.8. Synchronization Primitives
- 3.9. EGLImage Specification and Management
- 3.10. Posting the Color Buffer
  - `eglSwapBuffers` posts color buffer to a native window.  It has no effect
    on pbuffer or pixmap.  Its effect happens at a finite time later (e.g.,
    next vsync).
- 3.11. Obtaining Function Pointers
- 3.12. Releasing Thread State

## Chapter 4. Extending EGL

## Chapter 5. EGL Versions, Header Files, and Enumerants

- 5.1. Header Files
- 5.2. Compile-Time Version Detection
- 5.3. Enumerant Values and Header Portability

## Chapter 6. Glossary
