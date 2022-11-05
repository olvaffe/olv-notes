Android Emulator
================

## libOpenglRenderer

- `libOpenglRender.so` has
  - `initLibrary`
  - `setStreamMode`
  - `initOpenGLRenderer`
  - `createOpenGLSubwindow`
  - `destroyOpenGLSubwindow`
  - `repaintOpenGLDisplay`
  - `stopOpenGLRenderer`
- When qemu is run with `-gpu on`,
  - it will call `android_initOpenglesEmulation` to load `libOpenglRender.so`
    and look up its symbols.  `initLibrary` is then called, and
    `setStreamMode` is called to set the stream mode to `STREAM_MODE_UNIX`
  - If that succeeds, in `android_startOpenglesRenderer`, `initOpenGLRenderer`
    is called with the AVD's resolution and port 22468.  `qemu.gles=1` is
    passsed to the kernel.
  - An OpenGL subwindow occupies the screen area of the skin.  When the skin
    is resized or rotated, it calls `destroyOpenGLSubwindow` to destroy the
    subwindow and `createOpenGLSubwindow` to create a new one.
  - when the skin needs to repaint itself, it calls `repaintOpenGLDisplay` to
    repaint the screen area.
- `initLibrary`
  - loads `getenv("ANDROID_EGL_LIB")` or `libEGL_translator.so` to initialize
    EGL dispatch table, `s_egl`
  - loads `getenv("ANDROID_GLESv1_LIB")` or `libGLES_CM_translator.so` to
    initialize GLESv1 dispatch table, `s_gl`
  - optionally, loads `getenv("ANDROID_GLESv2_LIB")` or
    `libGLES_V2_translator.so` to initialize GLESv2 decoder context, `s_gl2`
- `setStreamMode` sets `gRendererStreamMode`
- `initOpenGLRenderer` initializes `Framebuffer` and `RenderServer`
  - `Framebuffer::create` initializes EGL with `EGL_DEFAULT_DISPLAY`.  It
    creates two EGL contexts, `m_eglContext` and `m_pbufContext`, in the same
    share group.  A dummy pbuffer, `m_pbufSurface`, is also created.
    - `bind_locked` makes `m_pbufContext` current
    - it then checks for `GL_OES_EGL_image`, `EGL_KHR_gl_texture_2D_image`,
      and `EGL_KHR_gl_renderbuffer_image`
    - it calls `FBConfig::initConfigList` to convert `EGLConfig`s to
      `FBConfig`s
  - `RenderServer::Main` is run by another thread.  It listens for
    connections, and creates a `RenderThread` for each connection.
- `createOpenGLSubwindow` makes FrameBuffer creates an X11 subwindow on
  the emulator skin's window.  An EGL window surface is also created for the
  subwindow.
- `repaintOpenGLDisplay` makes the subwindow the current draw surface.  It
  then paints the last posted ColorBuffer to the subwindow.
- There is a thread running `RenderThread::Main` for each process in the
  emulator that draws
  - A render thread has per-thread `RenderThreadInfo`.  It has `GLDecoder` and
    `GL2Decoder` that are initialized to use the GLES libraries loaded in
    `initLibrary`
  - It runs an infinite loop to decode input and dispatch to GLESv1, GLESv2,
    or RenderControl
- RenderControl decoder context is initialized with `initRenderControlContext`
  - every function is ultimately dispatched to FrameBuffer
  - half of the member functions are just mapped to EGL functions, using
    FrameBuffer's EGLDisplay
  - A `WindowSurface` is mapped to a pbuffer surface
  - A `ColorBuffer` has
    - two 2D textures, `m_tex` and `m_blitTex` in `FrameBuffer::m_pbufContext`
    - two EGLImages are created from the textures
    - An FBO created on demand for `m_tex` to attach to.  Whe a window
      flushes, it calls `glCopyTexImage2D` to copy from its pbuffer to
      `m_blitTex`.  FrameBuffer's context then copies from `m_blitTex` to
      `m_tex`
    - When to the color buffer is bound as texture/renderbuffer, ...
  - A `WindowSurface` gets a `ColorBuffer` for rendering
    - when the window is flushed, the contents of the pbuffer of the window
      are copied to the attached color buffer
  - A `ColorBuffer` can be posted.  Posting renders `m_mtex` of the
    ColorBuffer to FrameBuffer's subwindow.

## `libEGL_emulation`

- An `EGLThreadInfo` has `currentContext`, `hostConn`, and `eglError`
- A `HostConnection` has
  - `renderControl_encoder_context_t` to implement EGL
  - `GLEncoder` to implement GLESv1
  - `GL2Encoder` to implement GLESv2
- `libEGL_emulation` defines `EGLClient_eglInterface` to be passed to GLESv1's and
  GLESv2's `init_emul_gles`.  `init_emul_gles` returns
  `EGLClient_glesInterface`.
  - `EGLClient_eglInterface` defines methods to `getThreadInfo` and
    `getGLString`
  - `EGLClient_glesInterface` defines methods to initialize the library,
    GetProcAddress, and to `glFinish` the library
- `eglCreateWindowSurface` becomes `rcEnc->rcCreateWindowSurface`
- `eglCreatePbufferSurface` becomes `rcEnc->rcCreateWindowSurface` and
  `rcEnc->rcCreateColorBuffer`
- `eglCreateContext` becomes `rcEnc->rcCreateContext`
  - an EGL context has `GLClientState` for client states
- `eglMakeCurrent` becomes `rcEnc->rcMakeCurrent`
  - If it is GLESv1 context, `GLEncoder` is set to use the `GLClientState` of
    the context.  Similar for GLESv2.
  - `init` of the `EGLClient_glesInterface` is called
  - For a window surface, a buffer is dequeued and set as the color buffer of
    the surface through `rcEnc->rcSetWindowColorBuffer`
- `eglSwapBuffer` becomes `rcEnc->rcFlushWindowColorBuffer` for window surfaces.
  The color buffer is enqueued, and the next buffer is dequeued and set as the
  color buffer.  Finally, the host connection is flushed.
- `eglCreateImageKHR` returns the `android_native_buffer_t` passed to it.

## `libGLESv1_CM_emulation` and `libGLESv2_emulation`

- both libraries define `init_emul_gles(eglIface)`, called when a context of the
  api is made current for the first time.
- In GLESv1, `GET_CONTEXT` returns `gl_client_context_t`
  - `gl_ftable.h` generated by `emugen` defines a table that maps function names
    to functions.  It is used for GetProcAddress.
  - `gl_entry.cpp` generated by `emugen` defines all entrypoints.  Each
    entrypoint calls `GET_CONTEXT` and uses the context as the dispatch table.
  - `gl_enc.cpp` generated by `emugen` defines the dispatch table.  The
    functions encode and write the calls to the host connection.
  - These functions in the dispatch table are overridden by `GLEncoder`
    - `glFlush` flushes also the host connection
    - `glPixelStorei`, `glVertexPointer`, `glNormalPointer`, `glColorPointer`,
      `glPointSizePointerOES`, `glClientActiveTexture`, `glTexCoordPointer`,
      `glMatrixIndexPointerOES`, `glWeightPointerOES` updates also the state
      in `GLClientState`
    - `glGetIntegerv`, `glGetFloatv`, `glGetBooleanv`, `glGetFixedv` gives
      `GL_COMPRESSED_TEXTURE_FORMATS` special treatment and look up in
      `GLClientState` first
    - `glGetPointerv`, `glEnableClientState`, `glDisableClientState` uses
      `GLClientState` instead
    - `glIsEnabled` uses `GLClientState` first.
    - `glGetError` checks for local errors first
    - `glGetString` returns local strings
    - `glFinish` calls `ctx->glFinishRoundTrip`
    - `glBindBuffer` updates also `GLClientState`
    - `glDrawArrays` and `glDrawElements` sends the vertex attributes or
      just offsets depending on whether VBO is used.
  - These functions in the dispatch table are overridden by GLESv1
    - `glGetString` uses `getGLString` providied by EGL
    - `glEGLImageTargetTexture2DOES` calls `rcEnc->rcBindTexture`

## `gralloc.goldfish`

- `rcEnc->rcGetFBParam` is called for fb info
- A buffer consisits of
  - An ashmem fd is any of the SW flag is set
  - A host handle, returned by `rcEnc->rcCreateColorBuffer` is any of the HW
    flag is set
- On lock, `rcEnc->rcColorBufferCacheFlush` is called if there is a host handle
  - what for?
- On unlock, `rcEnc->rcUpdateColorBuffer` is called to copy the contents in
  ashmem fd to host handle
- On post, `rcEnc->rcFBPost` is called

## RenderControl Encoder

- see `opengl/system/renderControl_enc/README`

## Host Framebuffer

- It is a singleton initialized by `FrameBuffer::initialize`
- In `init_egl_dispatch`, it loads `libEGL_translator.so` and initializes the
  dispatch table
  - Same for `libGLES_CM_translator.so` and `libGLES_V2_translator.so`
- It then initializes EGL using the loaded module
- A subwindow is created to cover a subrectangle of the qemu window
- A GLESv1 context is created, called the renderer's context

## Renderer Process

- It runs on the host
- It renders to a subwindow of the qemu window
- For each connection (GL thread), a `RenderThread` is created on behalf of the
  client GL thread
  - There is thread-local `RenderThreadInfo`, storing `currContext`,
    `currDrawSurf`, `currReadSurf`, `m_glDec`, and `m_gl2Dec`
  - `GLDecoder`'s dispatch table is initialized with functions from the host GLESv1
    - several pseudo GL functions such as `glVertexPointerOffset` are overridden
      with local versions
  - Same to `GL2Decoder`
  - the thread's `renderControl_decoder_context_t` is also initialized.  Most
    functions will simply be dispatched into the `Framebuffer`
- `RenderContext` represents a client `EGLContext`
  - it has a host `EGLContext`
- `ColorBuffer` represents a client buffer object
  - it has a texture object in the renderer's context and an EGLImage of the
    texture object when the host EGL supports `EGL_KHR_gl_texture_2D_image`
  - there is also an FBO that the texture object can attach to.  It is used for
    the pbuffer of a `WindowSurface` to blit to
- `WindowSurface` represents a client `EGLSurface`
  - it has a host EGL pbuffer

## Examples

- 2D rendering
  - an app asks for a surface
    - a texture object and an EGLImage for it is created on the host
    - a ashmem is created on the client
  - the surface is mapped for CPU access
    - the ashmem is mapped
  - the surface is unmapped
    - the texture object is updated with `glTexSubImage`
- 3D rendering
  - a window surface is created for a native window
    - a pbuffer surface is created on the host
  - a context is created
    - a context is created on the host
  - the context and window are made current
    - the context and the pbuffer on the host are made current
    - one buffer of the client window is dequeued and is set as the attached
      color buffer of the host `WindowSurface`
  - rendering
    - the pbuffer is rendered to
  - swap buffers
    - the renderer's context is made current
    - the FBO for the attached color buffer is craeted
    - the pbuffer is bound as the texture
    - the pbuffer is blit to the attached color buffer
    - for a host that does not support bind-to-texture,
      - the pbuffer is made current with its original context
      - the pixel data is read using `glReadPixels`
      - the pixel data is downloaded to the color buffer using `glTexSubImage`
- Compositor
  - same as 3D rendering
  - binding a client `EGLImage`
    - bind the host `EGLImage` of the texture object of the color buffer
  - swap buffers
    - after the steps above
    - the renderer's context and subwindow are made current
    - the color buffer is drawn to the renderer's subwindow.
    - a host swap buffers is called

## Sync issues

- How renderer process works when guest has SurfaceFlinger and 2D apps
  - 2D apps `glTexSubImage` to texture objects in the renderer's context
  - SF thread has its own texture objects for each 2D apps.  The front buffers of
    the 2D apps are streamed to the texture objects through `EGLImageKHR`.
  - SF renders to a pbuffer
  - The contents of the pbuffer are read (`glReadPixels`), and used to update
    (`glTexSubImage`) SF's color buffer
  - On post, the SF's color buffer, a host texture, is rendered on the qemu
    subwindow
  - No sync issues
- How renderer process works when guest has SurfaceFlinger and 3D apps
  - 3D apps renders to their pbuffers with their contexts on the host
  - when swap buffers, the pbuffers are read in their contexts and the color
    buffers are updated in the renderer's context.
  - the rest should be te same as the 2D apps case
  - No sync issues

## Translators

- These are EGL/GLESv1/GLESv2 implementations based on GL 2.0

## EGL Translator

- Types
  - `EGLNativeDisplayType` the native display as known by guest
  - `EGLNativeInternalDisplayType` the native display as known by the host
    - OS dependent
  - `EglDisplay` is `EGLDisplay`, created from an internal display
- `eglInitialize`
  - it loads `libGLES_CM_translator.so` and looks up `__translator_getIfaces`
  - similar for GLESv2
  - it calls `glXGetFBConfigs` and creates a list of `EglConfig`
- Thread info
  - `EglThreadInfo`: EGL info for a thread
  - `ThreadInfo`: GLES info for a thread
- `EGLiface`
  - called from GLES
  - `getGLESContext` returns the current `GLEScontext`
  - `eglAttachEGLImage` maps an `EGLImageKHR` to an `EglImage` and remembers the
    mapping in the current `EglContext`.  The `EglImage` is returned.
  - `eglDetachEGLImage` undoes above
- `EglContext`
  - if `eglCreateContext` is called without a shared context, `createShareGroup`
    is called to create a new share group
  - otherwise, `attachShareGroup` is called
- `GLESiface`
  - called from EGL
  - `createGLESContext` is called in `eglCreateContext`
  - `initContext` is called when a 
  - `deleteGLESContext`
  - `flush`
  - `finish`
  - `setShareGroup`
  - `getProcAddress`
- `eglCreateWindowSurface` or `eglCreatePbufferSurface` is supported and straightforward
- `eglCreateContext`
  - creates an `GLEScontext`
  - creates an `GLXContext`, using the global shared context as the shared
    context.
  - creates an `EglContext` around the new `GLEScontext` and `GLXContext`
- `eglMakeCurrent` and `eglSwapBuffers` are straightforward
- `eglCreateImageKHR` needs to look up an texture object id
  - the shared group is used.

## EGLImage

- Scenario
  - a texture object is created in context A
  - an EGLImage is created from the texture object
  - another texture object is created in context B, with the EGLImage as the
    source
- Requirements
  - given a texture object id, EGL must know how to look it up
  - given an EGLImage, GLES must know how to look it up
  - the texture object must be shared by contexts A and B, but the texture
    object id must not (unless A and B are shared)
- HOWTO
  - because the texture object must be shareable by all contexts, the first
    `GLXContext` created will be shared with all `GLXContext`.  See
    `eglCreateContext`
  - as a result, the real context sharing also works
  - and as a result, namespaces need special care.  Two contexts not supposed to
    be shared are now shared.  They should use different namespaces.
    - every object has a local name.  It is the name known to the
      context and its (real) shared contexts.
    - an object also has a global name.  It is the name known by all
      `GLXContext`
    - The local name is always mapped to a global name before used
  - EGL knows how to look up the texture object using its local name.
    Additional info must be associated with the local name so that EGL knows the
    width and height of the texture object
