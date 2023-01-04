Mesa and EGL
============

## Thread Safety

- the spec states "EGL and its client APIs must be threadsafe. Interrupt
  routines may not share a context with their main thread."
  - I guess the second sentence is about client apis, where a context can only
    be current to a single thread
  - it seems valid to call, for example, `eglInitialize` and `eglTerminate` at
    the same time.  After both calls return, the display may be initialized or
    terminated depending on which happens first.
- EGL functions that does not take a `EGLDisplay`
  - these can use the "current thread info" tls trivially
    - `eglGetError`
    - `eglBindAPI`
    - `eglQueryAPI`
    - `eglGetCurrentContext`
    - `eglGetCurrentDisplay`
    - `eglGetCurrentSurface`
    - `eglReleaseThread`
      - this unbinds context/surfaces as well
  - these use the current context
    - `eglWaitClient`
    - `eglWaitGL`
    - `eglWaitNative`
  - these are standalone
    - `eglQueryString`
      - display is optional
    - `eglGetProcAddress`
    - `eglGetDisplay`
    - `eglGetPlatformDisplay`
- some other EGL functions
  - `eglInitialize`
  - `eglTerminate`
  - `eglMakeCurrent`
  - display resources that can be created/destroyed/manipulated
    - `EGLContext`
    - `EGLSurface`
    - `EGLSync`
    - `EGLImage`
- mesa implementation notes
  - there is a `_EGLThreadInfo` TLS
  - resources (contexts, surfaces, etc.) of a `EGLDisplay` are reference-counted
    - such that destroying a context that is still current does not lead to UAF

## EGL

- egl/main provides libEGL.so which implements EGL
- it dlopens what we call EGL driver here
- `egl/drivers/glx` provides `egl_glx.so`, which is an EGL driver for GLX.
- other drivers fail to build
- `gallium/winsys/egl_xlib` provides `egl_softpipe.so`, which is an EGL
  driver.  It works with whatever state trackers.

## Thread

- There are many displays, contexts, and surfaces.
  - every thread has the notion of current api, current context, and current
    error
  - `eglBindAPI` changes current api of calling thread.
    - Because there can be one current context for every api, it affects
      `eglCreateContext`, `eglGetCurrentContext`, `eglGetCurrentDisplay`,
      `eglGetCurrentSurface`, `eglMakeCurrent` (when its `ctx` parameter is
      `EGL_NO_CONTEXT`), `eglWaitClient`, or `eglWaitNative`.
  - `eglMakeCurrent` changes current context of calling thread.
    - The api of the context is given at creation time.
  - any EGL call might change current error
- `eglGetDisplay` to create displays
- `eglCreateContext` to create contexts of a display
- `eglCreateXxxSurface` to create surfaces of a display
- A thread can have at most one current context for each supported api
- A context can be current to at most one thread
- An initialized display can be used by other threads.

## Impl. Notes

- rendering APIs or client APIs mean OpenGL ES or OpenVG.
- native window systems or native rendering APIs mean X Window
- all EGL objects are associated with an EGLDisplay.
- Two EGLContext might share a EGLSurface.  NOT TRUE.
- (2.2) A context can be used with any compatible surface.  They are compatible if
  - They support the same type of color buffer
  - They have color buffers and ancillary buffers of the same depth
  - The surface was created with respect to an EGLConfig supporting client API
    rendering of the same type as the API type of the context
    - surfaces and contextes are created with a config
  - They were created with respect to the same EGLDisplay
- each thread can have at most one current rendering context for each supported
  client API
- back buffered rendering is used by window and pbuffer surfaces.  The buffer is
  allocated and owned by EGL; pixmap surface must be single buffered (2.2.2).
  OpenGL supports single buffered window surface.
- `eglSwapBuffers` must respond to native window size changes
- Two contexts of the same API type may share state (2.4)
- If the system is suspended or something, `eglSwapBuffers`, `eglMakeCurrent`,
  etc. will return false with `EGL_CONTEXT_LOST`.  client should
  `eglDestroyContext` all contexts and recreate new ones.  Surfaces need not be
  recreated. (2.6)
- binding a bound context generates `EGL_BAD_ACCESS`. (3.1)
- multiple calls to `eglGetDisplay` with the same display id return the same
  display handle (3.2).  displays may be shared by threads.
- `eglTerminate` marks _all_ egl resources for deletion.  Resources that are
  current are destroyed only when they are no longer current.
- A config consists of attributes.  A surface or context is also created with a
  config.  But do not confuse config attributes with surface or context
  attributes. (3.4)
  - config attrs are retrieved by `eglGetConfigAttrib`
  - surface attrs are retrieved by `eglQuerySurface` and set by
    `eglSurfaceAttrib` (3.5)
  - context attrs are retrieved by `eglQueryContext`
- If a config has `EGL_SWAP_BEHAVIOR_PRESERVED_BIT` in its `EGL_SURFACE_TYPE`,
  `EGL_SWAP_BEHAVIOR` attr of a surface created may be specified to preserve
  buffer contents (3.5.6)
- Rendering to textures is supported only by OpenGL ES and pbuffer surfaces.
  (3.6)
- Rendering to textures might be deprecated by FBO (3.6.3)
- At most one context for each supported client APIs may be current to a thead
  at any given time. Though, no two of the current contexts can draw to the same
  surface (3.7).
  - Only one of OpenGL and OpenGL ES can be current at one time.

## EGLImage

- In client apis, EGLImage can be used as a texture
- In EGL, native pixmap or client api texture can be turned into a EGLImage.
  - A OpenVG VGImage can be turned into a EGLImage and used as a 2D texture.
- the image source `buffer` in `eglCreateImageKHR` cannot
  - has an offscreen buffer bound to it (by `eglBindTexImage`)
  - be bound to an offscreen buffer (by `eglCreatePbufferFromClientBuffer`)
  - be an image target in some client context
  - however, the `buffer` may has subobjects (different level of a texture,
    cubemaps).  If a image has some subobj as its source, it is possible to
    create another image using a different subobj.  See Q5 of the spec.
- Similarly, `eglCreatePbufferFromClientBuffer` cannot be called on a buffer
  that is an image sibling.
- `eglCreateImageKHR` takes a context so that it knows to which context does
  that buffer belongs.
  - e.g., buffer can be the name of a 2d texture object.  `_mesa_lookup_texture`
    takes a context and a name to perform the lookup.
- As long as there are image targets, or `EGLImage` exists, the buffer of image
  source remains allocated.
  - destroying an image by `eglDestroyImageKHR` does not destroy its buffer or
    siblings.
  - That is, image source, image targets, and the image all reference the
    buffer.
- The modifications to the contents (such as `glTexSubImage`) of a image target
  in client apis reach the buffer of the image source.  That is, all image
  targets will see the changes.
- Contrary to modifications, respecifications (change of the format or size of
  the image target, or `glTexImage`) usually result in orphaning the image.
  New buffer might be allocated, and the contents of the orphaned image might be
  copied to the new buffer.
  - See also Q6 of the spec.
- The rules of modifications and respecifications apply also to image source.
- What would happen to the images when the display is terminated?
- <http://groups.google.com/group/android-porting/browse_thread/thread/fce70ae42e7dd87b?fwc=1>
- `EGLImageKHR` should be reference counted?
  - It is used in EGL, and client apis.
  - Should `eglDestroyImageKHR` free the data structure immediately?
  - how does a client api know an image?  should state tracker include
    `eglimage.h`?  apparently not.
- Given a native pixmap, we need to decide a compatible EGLConfig
  - a config that hsa the same `EGL_NATIVE_VISUAL_TYPE` as pixmap is.
  - ask backend the visual type of a pixmap.  compare with all configs.  cache
    the result.
  - for `EGL_MATCH_NATIVE_PIXMAP`, the same can be done ,without caching.

## Next Step: Thread Support

- Driver: `egl_softpipe.so`
  - All drawings happen locally.  It is put image to x server.
- API: OpenGL
- `progs/egl/xeglthreads.c` for test program
- `src/egl/main/`
  - `eglcurrent.c` for current objects (threads) management. in case a thread
    info failed to allocate, a read-only dummy thread info is used.
  - Terminate a display should destroy its contexts and surfaces.
  - `IsBound` should be `BoundTo` so that we can know which thread a context is
    bound to?
- `src/gallium/winsys/egl_xlib/` need fixes

## `egl_softpipe.so`

- In its `_eglMain`, it creates a `struct pipe_winsys` and a
  `struct pipe_screen`.
  - `softpipe_create_screen` is called on the winsys to create a pipe screen.
- In its `Initialize`, all X visuals are queried and turned into `EGLConfig`s.
- In its `Terminate`, nothing is done.
- In its `GetProcAddress`, `st_get_proc_address` is called.
- In its `CreateContext`,
  - `softpipe_create` is called on the pipe screen to create a pipe context.
  - the provided config is converted into a `__GLcontextModes` (`GLvisual`) and
    is passed to `st_create_context` to create a `struct st_context`.
- In its `DestroyContext`, `st_destroy_context` is called to destroy the st
  context.
- In its `CreateWindowSurface`,
  - `XCreateGC` is called to create an empty GC.
  - The `Window`'s visual is queried and stored.
  - `st_create_framebuffer` is called to create a `struct st_framebuffer` and
    `st_resize_framebuffer` is called to resize the framebuffer.
- In its `DestroyWindowSurface`, `st_unreference_framebuffer` is called to
  destroy the st framebuffer.
- In its `MakeCurrent`, `st_make_current` is called.
- Finally in its `SwapBuffers`,
  - `st_get_framebuffer_surface` is called to get the back left `struct
    pipe_surface` of the st framebuffer.
  - `st_notify_swapbuffers` is called to notify the st framebuffer.
  - Using the surface's visual and GC, an `XImage` is created to upload the
    contents to xserver.
  - The contents come from using pipe winsys's `buffer_map` to map the pipe
    surface's pipe texture's `struct pipe_buffer`.
- With these in hand, we know that
  - `egl_softpipe.so`, i.e. EGL, is a pipe winsys.
  - it chooses softpipe to create a pipe screen.
  - an egl context is associated with a pipe context and a st context.  The pipe
    context is created with the pipe screen while the st context is created with
    the pipe context.
  - an egl surface is associated with a st framebuffer.
  - the st framebuffer has many pipe surfaces (renderbuffers).  Every pipe
    surface has a pipe texture.  The pipe texture has a pipe buffer.  It relies
    on pipe driver's specific way to turn a pipe texture to a pipe buffer.
- A look at `egl_softpipe.so`'s winsys shows that a winsy is supposed to
  - create/destroy `struct pipe_buffer`s.
    - It comes in three flavors: normal, user, and surface.
    - See `struct pipe_winsys`.
  - flush the buffers for display.

## Pbuffer

- Pbuffer is used for offscreen accelerated rendering (3.5.2)
  - And to be bound to an OpenGL ES texture object.
  - Pbuffer is owned by EGL
  - When created, `EGL_WIDTH` and `EGL_HEIGHT` give the width and heght
  - `EGL_TEXTURE_FORMAT` gives the format of the texture if created
  - Pbuffer may also be created from `eglCreatePbufferFromClientBuffer` (3.5.3)
    - Pbuffer will reference the color buffer of the client buffer
    - It is up to the implementation to allow such pbuffer to be used as a
      texture.  Or other constraints.
    - `EGL_BAD_ACCESS` if the client buffer is already used by another pbuffer
    - `EGL_BAD_ACCESS` if the client buffer is used by the client API itself at
      `eglMakeCurrent` time (3.7.3)
- When an implementation supports
  - only OpenGL ES, pbuffer allows render-to-texture. (FBO is preferred).
  - only OpenVG, pbuffer allows render-to-image.
  - both OpenGL ES and OpenVG, pbuffer allows also
    - render-opengl-es-to-image
    - use-image-as-texture
- How to EGLImage?
  - EGL can create an EGLImage from VGImage or OpenGL ES texture object
  - OpenVG can create a VGImage from an EGLImage
  - OpenGL ES can use an EGLImage as a texture or as a FBO renderbuffer.
    - The former uses contents of the EGLImage
    - The latter draws to the EGLImage

## `eglBindTexImage`

- Use `_mesa_select_tex_object` to find the texture object, given the texture
  target.
  - There are 1d, 2d, 3d, and cube texture objects.  The texture targets 1d,
    2d, and 3d are mapped to respective texture objects.  The target cube or any
    of the six faces of the cube gives the cube texture object.
- Flush and find the color buffer of the pbuffer.
- Use `_mesa_get_tex_image` to find the texture image of the texture object at
  base level.
- Release previous texels for all levels
- Make the texture image reference the color buffer.
  - for buffer object based texture image, `gl_texture_image->Data` is NULL.
- Related `EGLConfig` attributes
  - `EGL_SURFACE_TYPE` must has `EGL_PBUFFER_BIT` set.
  - `EGL_BIND_TO_TEXTURE_RGB` and `EGL_BIND_TO_TEXTURE_RGBA` should be true to
    be bindable to RGB or RGBA textures.
- Related pbuffer attributes
  - `EGL_TEXTURE_FORMAT` gives `EGL_TEXTURE_RGB` or `EGL_TEXTURE_RGBA`
  - `EGL_TEXTURE_TARGET` must be `EGL_TEXTURE_2D`.
  - `EGL_MIPMAP_TEXTURE` is a boolean whether mipmap should be generated.
- For glx, the idea of mipmap is
  - if config says true, an additional space is allocated when creating the
    pixmap
  - `GenerateMipmapEXT` would generate the mipmaps
- The rationale
  - A texture object is buffer based or pixmap/pbuffer based
    - No mix.
  - Each texture image has its storage, from a buffer or pixmap.
    - the buffer may be allocated per-image, or per-object.  It is
      implementation details.
  - A pixmap may be bound to multiple txture objects.
    - allow binding to multiple texture images is possible, but does not make
      sense.
    - a release releases all bindings.
  - Completeness of a texture object has nothing to do with its contents.
    - See `_mesa_test_texobj_completeness`.
    - If a texture object is complete, all images must have width and height
      greater than zero.  It implies a buffer if the texture object is buffer
      based.  But the contents can be undefined.
    - If it is pixmap based, release the pixmap should make it incomplete.
- Now have a look at gallium
  - The stObjs to be set to pipe driver are decided.  They are complete.
  - Every stObj may have a pipe texutre.
  - Every stImage of a stObj may have a pipe texture.
  - It is stObj's pipe texture that is set to pipe driver.
  - The texels of an image might store in its pipe texture or a private buffer.
    - They should be uploaded to stObj's pipe texture.
    - It is possible that the pipe texture is the same as stObj's.  No upload in
      this case.
    - This step is called finalization.
  - One issue here: texture-from-pixmap.
    - The reason an image has a pipe texture different from the stObj's is
      texture-from-pixmap.
    - We want to avoid copy as possible as we can.
    - How? Replace stObj's pipe texture by stImage's.

## pbuffer bottom up

- When a pbuffer is bound, the original texels of a texture object should be
  freed, if they are private buffer based.  That is, a texture object is either
  private buffer based or pbuffer based.
- In `st_set_teximage`, we want to free the texels of a texture object first.
  How?
- Have a look at `_mesa_BindTexture` first.
  - It flushes vertices and marks `_NEW_TEXTURE`.
- Let's see what would happen when, say, `glDrawArrays` is called
  - It is dispatched to `vbo_exec_DrawArrays`.
  - `_mesa_update_state` is called to update the states first
  - `update_texture_state` is called to, among others,
    `_mesa_test_texobj_completeness` the newly bound texture if it is not
    complete yet, and makes the texture unit reference the new texture object.
  - In the end of state update, `st_invalidate_state` is called.
  - At this point, mesa context was marked dirty and have been brought
    up-to-date.  st context is marked dirty only.  pipe driver is unaware of
    these changes.
  - Finally, st is entered.  It is `st_draw_vbo` that is called.
  - Similarly, `st_validate_state` is called to validate st context first.
    It calls `update_textures` and tells pipe driver the new texture objects.
    For softpipe, `softpipe_set_sampler_textures` is called.  Before anything,
    it calls `draw_flush` to flush the pipeline.  At this point, the pipe driver
    is unaware of the new texture object.  `draw_flush` flushes the pipeline so
    that the buffered vertices are rasterized with old texture object.  It then
    installs the new texture object.
- In summary, there are 5 places states are remembered.  The state the user
  thinks, the state of mesa context, the state of st context, the state of the
  pipe context, and the state of the hw.
  - The first is changed immediately upon `glXxxOoo` and the mesa context is
    marked dirty (through `ctx->NewState`).
  - It propogates to the mesa context when `_mesa_update_state` is called, and
    the st context is marked dirty (through `st_invalidate_state`).
  - It propogates to st context when `st_validate_state` is called, and the pipe
    context is marked dirty (through `bind_vs_state`, etc.).
  - It propogates to pipe context when `i915_update_derived` is called, and the
    new hw state is calculated.
  - It propogates to hw when `i915_emit_hardware_state` is called.

## Some examples

- `_mesa_create_context`, `_mesa_notifySwapBuffers` and `_mesa_make_current`
- `_mesa_initialize_framebuffer` and `_mesa_create_framebuffer`.
  - Contrary to `_mesa_new_framebuffer`, which is a driver call to support FBO.
- No OpenGL API calls the functions
- They are used by the window API like GLX or EGL.  Or, the driver.
- Functions like `eglMakeCurrent` or `eglSwapBuffers` reach the driver first.
  The driver then calls the respective mesa functions.
  - OpenGL API reaches mesa functions first, which then call into the driver.
- Because of the fundamental difference, there is no need for
  `driver->MakeCurrent` or `driver->NotifySwapBuffers`.
- Now, what is `eglBindTexImage`?
  - It should reach the driver first, and then the mesa function
    `_mesa_bindTexImage`.
  - There is no need to add `driver->BindTexImage`.

## double buffer

- A GL context's default framebuffer object has many buffers
  - front, back, left, right, aux, depth, stenctil, ...
  - not all buffers are always available; a single buffered context has no back
    buffer, for example
  - it may choose what buffer(s) to draw to
  - the backing storage of the buffers are owned by the window system or EGL.
- GLX and buffers
  - the buffers of a GLX drawable is described by the fbconfig
  - how buffers of a drawable map to buffers of a fbobj is not well-defined.
  - `glXSwapBuffers` is no-op on pixmap and single-buffered drawable
    - as a corollary, the left front buffer of the fbobj must be the
      underlying buffer of the single-buffered drawable or pixmap.
    - otherwise, there is no way to display the contents.
    - further, `glFlush` might cause flipping because the front buffer might be
      a fake one in a composited environment
- EGL and buffers
  - `eglSwapBuffers` is no-op on pixmap, pbuffer, and single-bufferred window.
  - window surface is required to be double-buffered when used with OpenGL ES
- pbuffer and pixmap are single-bufferd.  pbuffer renders to `EGL_BACK_BUFFER`
  while pixmap renders to the native buffer.
- egl + opengl es with window surface is always double-buffered.
  - opengl es supports single-bufferd.
  - but double-buffered is a requirement by egl
- For a given native visual,
  - if it is double-buffered, it cannot have pbuffer or pixmap bits set.
    When giving buffers to a window surface, native buffer is given to
    `FRONT_LEFT` and back buffer is given to `BACK_LEFT`.
    - querying the render buffer of a context binding to such window surface
      always returns `EGL_BACK_BUFFER`.
    - If OpenGL draws to front left directly, calling `eglSwapBuffers` gives
      undefined contents.  This is the correct behavior.
  - if it is single-buffered, the native buffer is given to pixmap and window
    surface.  a back buffer is allocated for pbuffer surface.
    - it cannot have opengl es bit set in `EGL_RENDERABLE_TYPE`.  opengl es is
      required to be double-buffered in egl.
    - querying thr render buffer of a context binding to such window surface
      always returns `EGL_SINGLE_BUFFER`.
- the buffering of a context is determined at initialization time, by its
  config.  It might as well say that, the result of querying `EGL_RENDER_BUFFER`
  to a context is totally decided by the context itself.
- It is also possible to ignore all double-buffered visuals.  And when a buffer
  for `FRONT_LEFT` is asked,
  - returns native buffer if the surface is single-buffered window or pixmap.
  - returns a newly allocated buffer if the surface is back-buffered window or
    pbuffer.
  - it is sort of how `egl_android` should work.  But since the native buffer is
    constantly switching, there can be no single-bufferred window.

## natives

- We want both clients and the server can use libEGL
  - Biggest problem: for server, there is no native type
  - Need another view: for server, they use different native types
    - client native types
    - server native types
  - EGL native types: client or server native types can be wrapped into
- Native window
  - for client, it sees ClientNativeWindowType
  - for server, it sees ServerNativeWindowType
  - for EGL, it needs EGLNativeWindowType
  - there should be a way to wrap both client and server native window in a egl native window
- Sort of the Android way
- Roles
  - In composition world, a client native window is _NOT_ a server native window
  - A screen/connector/monitor _IS_ a server native windows
  - That is, a server native window is a scan-out buffer
  - What is the role of a client window in server?
- Pbuffer way
  - A client native window is a pbuffer in server
  - The pbuffers are composited to the server native window(s)
  - However, a pbuffer is an EGL owned resource
  - There is no access in the server to the underlying buffer of a pbuffer
  - How to pass the buffer to the client?  maybe this won't work
- Buffer way
  - Everything is, after all, just buffers
  - Buffers that can cross the boundary of processes
  - A client native window is a buffer in server
  - A buffer in server should be made a server native pixmap. It cannot be
    a window because a scan-out buffer is a server native window.
  - If it goes further, there is less native types than it seems like
    - server native window: scan-out buffer
    - server native pixmap: buffer
    - client native window: a server-defined window, backed by a buffer
    - client native pixmap: buffer
    - egl native window: something that wraps server and client native window
    - egl native pixmap: same like egl native window, or, could just be buffer
    - finally, there is pbuffer
- EGLImage
  - A buffer is a server native pixmap.
  - When it is wrapped in an egl native pixmap, a EGLImage can be created
  - Zero-copy texturing can be done.
- Native display
  - Should there be client native display, server native display, and egl native
    display?  Is it over complicated?
  - Either this way, or something out-of-band.
  - What if there is only one native display, fd?  Server passes a master fd.
    Client uses default display, which creates a new fd.  The fd will be
    authenticated, unless it is already a master.  This is out-of-band.

## Android and native types

- double buffered
  - In android, a client native window is a Surface.
  - A Surface is a Layer in the server.
  - It has two buffers!
  - A simple way is to view a client native window as two server native
    pixmaps.  two EGLImage are required.
  - Or, it is possible to view the two buffers together as a server native
    pixmap.  When the EGLImage is bound, the pixmap is locked.  No swapping.


## Lifetime

- Surfaces and contexts are resources of a display
- Display resources can be released by calling the destroy functions.  They are
  also released when a display is terminated
- However, current thread holds references to the current contexts; Current
  contexts and current surfaces hold references to each other; The resources
  have pointers pointing back to the display
  - the resources live longer than the display
  - One can get acess to `_EGLDisplay`, `_EGLContext`, and `_EGLSurface` through
    current thread info.  They cannot be freed.  `_EGLDisplay->DriverData` might
    be freed earlier when the display is terminated.  The main library does not
    care.

## Surfaces

- Given a window system, there may be windows and pixmaps
  - windows are resizable on-screen objects
  - pixmaps are fixed size off-screen objects
  - A window or pixmap may consist of one or more color buffers
    - one of them is the front buffer;  the rest are back buffers
    - there may also be right color buffers for stereoscopic rendering
  - there may be native rendering mechanism
    - usually only can render to the front buffer
  - there may be a mechansim to manipulate buffers
    - a way to get the next one or more back buffers
    - optionally a way to get the front buffer
    - a way to make a back buffer front
      - buffer preserving or not (copy or flip)
      - wait for vsync or not (swap interval)
      - accurate timing (for AV sync)
    - optionally a way to copy a local buffer to front
      - e.g. copy pbuffer to pixmap
      - e.g. post a subregion of a back buffer
    - a way to sync with the native rendering mechanism
      - Usually the native rendering mechanism renders to the front buffer.  To
        be able mix it with GL rendering, the capability to get the front buffer
        is required.
- Under EGL,
  - there is pbuffer which consists of a single local color buffer
    - Under GLX, a pbuffer can have multiple local color buffers as pixmaps
  - pixmap surfaces must support getting the front buffer so that mixing native
    and GL rendering works
  - GLES does not support rendering to the front buffer of a window surface
    - meaningless restriction by EGL
  - GL requires the front buffer to be available
    - required by GL spec

## dma-buf

- `eglQueryDmaBufFormatsEXT`
  - `dri2_query_dma_buf_formats` (in gallium dri, not the one in egl dri)
    loops over `dri2_format_table` and returns formats that can be supported
    as `PIPE_BIND_RENDER_TARGET` or `PIPE_BIND_SAMPLER_VIEW` natively
  - it also uses `dri2_yuv_dma_buf_supported` for planar formats
- `eglQueryDmaBufModifiersEXT`
  - `dri2_query_dma_buf_modifiers` (in gallium dri, not the one in egl dri)
    gets the supported modifiers from the pipe driver
  - if no native sampling support and relies on `dri2_yuv_dma_buf_supported`,
    `external_only` is set to true
- `eglCreateImage(EGL_LINUX_DMA_BUF_EXT, DRM_FORMAT_NV12)`
  - `dri2_from_dma_bufs2` uses a `dri2_format_mapping` that has
    - `DRM_FORMAT_NV12`
    - `__DRI_IMAGE_FORMAT_NONE`
    - `PIPE_FORMAT_NV12`
    - plane count 2
      - `__DRI_IMAGE_FORMAT_R8`
      - `__DRI_IMAGE_FORMAT_GR88`
  - `dri2_create_image_from_winsys` imports 2 plane fds as 2 `pipe_resource`s
    - without native support, `use_lowered` is true and the two resoruces have
      formats `PIPE_FORMAT_R8_UNORM` and `PIPE_FORMAT_RG88_UNORM` respectively
    - with native support, the two resources have format `PIPE_FORMAT_NV12`
    - on freedreno, there is semi-native support and requires a format
      translation
      - the two resources have format `PIPE_FORMAT_R8_G8B8_420_UNORM`
