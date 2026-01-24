Android EGL / OpenGL ES
=======================

## Loader

- `Loader::open`
  - if `shouldUseAngle`, it loads `lib{EGL,GLESv1_CM,GLESv2}_angle.so`
  - if `shouldUseNativeDriver`, it loads
    `lib{EGL,GLESv1_CM,GLESv2}_<name>.so`, where `<name>` is from
    `ro.hardware.egl`
  - else, it tries `<name>` from `HAL_SUBNAME_KEY_PROPERTIES` array
    - `persist.graphics.egl`
    - `ro.hardware.egl`
    - `ro.board.platform`
  - else, it tries non-suffixed `lib{EGL,GLESv1_CM,GLESv2}.so`

## Layers

- `eglGetDisplay` calls `egl_init_drivers`
  - `LayerLoader::getInstance` loads layers
    - `GraphicsEnv::getDebugLayersGLES` returns a list of layers (.so)
    - if no layer, `debug.gles.layers` is used instead
    - `GraphicsEnv::getLayerPaths` returns the paths to search
    - if debuggable, `/data/local/debug/gles` is searched too

## EGL (froyo)

- there is `Loader.cpp`
  - It reads `/system/lib/egl/egl.cfg` for the implementations
    - `egl.cfg` has `dpy impl tag` on each line
    - if `egl.cfg` is not found, `0 0 android` is assumed
  - given a display and an impl, `/system/lib/egl/lib<API>_<tag>.so` is loaded
    - it loads `EGL`, `GLESv1_CM`, and then `GLESv2`
    - Or, it loads the all-in-one `GLES`
  - after a module is loaded
    - `dlsym` or `eglGetProcAddress` is called to fill `egl_t` and `gl_hooks_t`
- `/system/lib/libEGL.so`
  - `egl_init_drivers` is called by functions that do not take an `EGLDisplay`:
    `eglGetDisplay`, `eglGetProcAddress`, `eglBindAPI`, `eglQueryAPI`
  - it loads module `0 0 android` for software impl and `0 1 <blah>` for hardware impl
  - as can be seen in `validate_display_config`, a config decides the
    implementation.  for functions that the impl cannot be decided, such as
    `eglCreateImageKHR`, all implementations are called
- app calls `eglFooBar` in `/system/lib/libEGL.so`
  - `eglFooBar` calls `cnx->egl.eglFooBar`
- app calls `glFooBar` in `/system/lib/libGLESv2.so`
  - `glFooBar` calls `getGlThreadSpecific()->gl.glFooBar`
  - `glEGLImageTargetTexture2DOES` is special in that it calls
    `egl_get_image_for_current_context` to convert `EGLImage` before calling the
    hook.
- the build system defines `BOARD_EGL_CFG` for the distributed `egl.cfg`
- `libagl` supports
  - `EGL_KHR_image_base`
  - `EGL_ANDROID_image_native_buffer`
    - `eglCreateImageKHR` with `EGL_NATIVE_BUFFER_ANDROID`, which is an
      `android_native_buffer_t`.
  - `EGL_ANDROID_swap_rectangle`
  - `EGL_ANDROID_get_render_buffer`
    - return the backing `android_native_buffer_t` of an `EGLSurface`

## Buffer Management (froyo)

- buffer allocation is done by SurfaceFlinger
- buffer allocation is requested by WindowManagerService
- an app asks WindowManagerService to ask SurfaceFlinger to allocate a buffer
- SurfaceFlinger creates layers
  - a layer has two `GraphicBuffer` as can be seen in `Layer::setBuffers`
  - a `GraphicBuffer` is allocated from gralloc module or `sw_gralloc_handle_t`
  - a layer also has a `SurfaceLayer` to accept requests from
    WindowManagerService
- WindowManagerService views a `Layer` as a `Surface`
  - it requests SurfaceFlinger to create a `Layer` through
    `ISurfaceFlingerClient::createSurface`.  The returned `ISurface` is wrapped
    in a local `SurfaceControl`, as thus a `Surface`
  - it calls `ISurface::requestBuffer` to get `GraphicBuffer`
    - A `GraphicBuffer` is a `Flattenable`.  It can be flatten and unflatten
      so that it can travel as a `Parcel`
- An app asks WindowManagerService to creat a java `Surface`
  - see `android_view_Surface.cpp`
- A `Surface` inherits `android_native_window_t`
  - it can be passed to `eglCreateWindowSurface`

## buffer usages

- A `Surface` is by default `GRALLOC_USAGE_HW_RENDER`
  - it can be set wit `setUsage`
  - when a surface is locked for CPU access,
    - `GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN` is set
- surfaceflinger layer
  - force the mode to sw read/write in secure mode
  - hw_texture is usually set for eglimage texturing
  - LayerDim, on msm7k, creates a sw write and hw texture buffer

## Native Types

- `eglCreateWindowSurface` takes a window of type `NativeWindowType`.  In
  android, a native window is a `Surface`.  Thus, a `Surface` wrapped in
  `EGLNativeWindowSurface` is passed to `eglCreateWindowSurface`.
- Note that framebuffer device is wrapped in `EGLDisplaySurface` and passed to
  `eglCreateWindowSurface`.
- Both `EGLDisplaySurface` and `EGLNativeWindowSurface` is a subclass of
  `EGLNativeWindowSurface` which is a subclass of `egl_native_window_t`, _THE_
  native window `GL` implementation expects.
- Similarly, a native pixmap is a `SkBitmap`.  It is wrapped in
  `egl_native_pixmap_t` and passed to `eglCreatePixmapSurface`.
- The implementation wraps all types of surfaces in a `GGLSurface`.  It is the
  natural type used by the impl. and is the object created when
  `eglCreatePbufferSurface` is called,

## `EGLNativeWindowSurface`

- inherits `EGLNativeSurface`, which inherits `egl_native_window_t`, which could
  be passed to `eglCreateWindowSurface`.
- is used by applications
- is ctor with a Surface
- as an `egl_native_window_t`,
  - `fd` is 0
  - `connect` means Surface->lock
  - `disconnect` means Surface->unlock
  - `swapBuffers` means Surface->unlockAndPost then Surface->lock
- comparing to `EGLDisplaySurface`, whose
  - `fd` is the framebuffer device
  - `connect` and `disconnect` is not set
  - `swapBuffers` copies back buffer to front buffer when no `PAGE_FLIP`; Or,
    simply flips becase `copyFrontToBack` has done the copy.

## `GLSurfaceView`

- When `start`, `EGLDisplay` is got and `eglCreateContext` is called.  It is
  done once because `eglCreateContext` is heavy/expensive.
- When a `SurfaceHolder` is got/changed, `eglCreateWindowSurface` is called on
  the surface to create `EGLSurface` and `eglMakeCurrent` is called.
- `Renderer` draws on the `EGLSurface` repeatedly in a loop.  Every time
  `onDrawFrame` returns, `eglSwapBuffers` is called.
- In view of an `egl_native_window_t`, the surface is locked most of the time
  because `connect` is called in `eglCreateWindowSurface`.

## `libGLESv1_CM.so`

- Provides GL entries.
- Any GL method goes through `getGlThreadSpecific` to find current `gl_hooks_t`.
  It might use implementation from `libhgl.so`, `libagl.so`, no context, etc,
  depending on current context.  When `GL_LOGGER` is defined, a call to
  `log_XXXX` is also made to log a message.
- A `gl_hooks_t` provides entries to GL functions, EGL functions, and
  extensions.

## `libEGL.so`

- The names of GL and EGL methods are built and stored in `gl_names` and
  `egl_names`.
- `libagl.so` and `libhgl.so` are represented by `egl_connection_t`.
- `load_driver` `dlopen`s the specified library.  It loops through every
  `gl_names` and `egl_names` to resolve the symbols and stores them in the
  specified `gl_hooks_t`.
- driver's `EGLContext` and `EGLSurface` are wrapped in `egl_context_t` and
  `egl_surface_t`.  They are in turn returned to users as `EGLContext` and
  `EGLSurface`.
- `eglGetDisplay` gets both sw and hw displays.
- `eglInitialize` initializes both sw and hw displays.
- `eglChooseConfig` chooses configs for both sw and hw displays.
- `eglCreateWindowSurface` and `eglCreateContext` decides which display to use.

## `GPUHardware`

- `request` makes the calling pid the owner of the GPU
  - If the GPU is owned for the first time, two `GPUAreaHeap` is created for
    `/dev/pmem_gpu?`.  A `GPURegisterHeap` is also created.
  - A client and the `mCurrentAllocator` is created.
  - If `request` is called with a `IGPUCallback` and a
    `ISurfaceComposer::gpu_info_t`, callback is also linked to death and heap
    infos are returned (to client for DRI!)
- `revoke` is called through `revokeNotification` when the client does not use gpu anymore
  - it signals mCondition
  - it calls `releaseLocked` to revoke current client's heaps
- if B tries to own gpu while A is the owner
  - `takeBackGPULocked` is called to notify A `gpuLost` and waits on mCondition
  - A calls `revoke`
  - `releaseLocked` is called to make sure A is not the owner anymore
- `SurfaceFlinger` calls `friendlyRevoke` when console released (chvt to another
  one) or remote calls `revokeGPU`.
- Comparison
  - `request(pid)` makes the pid current owner and returns the dealer
  - `request(...)` makes the pid current owner, registers the callback, and
    returns `gpu_info_t`
  - `revoke(pid)` (current owner) acks the revoke and releases the gpu
  - `friendlyRevoke(void)` notifies current owner and waits before releasing
     the gpu
  - `unconditionalRevoke(void)` releases the gpu

## Observations

- For `Surface`, an `EGLSurface` locks the surface until destroied.  When
  `swapBuffers`, `unlockAndPost` is immediately followed by `lock`.
  `bindDrawSurface` is then called to update buffer info (offset change caused
  by flipping).
- 

## Java bindings

- `javax.microedition.khronos.{egl,opengles}` describes one binding for EGL and
  GLES called JSR239
- `com.google.android.gles_jni` is an implementation of JSR239 using JNI
- `android.opengl` defines another Android-specific GLES binding, implemented
  also with JNI, as well as high-level constructs
  - all functions are static and are said to be faster
  - see commit 27f8002e591b5c579f75b2580183b5d1c4219cd4
