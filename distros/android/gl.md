Android OpenGL ES
=================

## EGL (froyo)

* there is `Loader.cpp`
  * It reads `/system/lib/egl/egl.cfg` for the implementations
    * `egl.cfg` has `dpy impl tag` on each line
    * if `egl.cfg` is not found, `0 0 android` is assumed
  * given a display and an impl, `/system/lib/egl/lib<API>_<tag>.so` is loaded
    * it loads `EGL`, `GLESv1_CM`, and then `GLESv2`
    * Or, it loads the all-in-one `GLES`
  * after a module is loaded
    * `dlsym` or `eglGetProcAddress` is called to fill `egl_t` and `gl_hooks_t`
* `/system/lib/libEGL.so`
  * `egl_init_drivers` is called by functions that do not take an `EGLDisplay`:
    `eglGetDisplay`, `eglGetProcAddress`, `eglBindAPI`, `eglQueryAPI`
  * it loads module `0 0 android` for software impl and `0 1 <blah>` for hardware impl
  * as can be seen in `validate_display_config`, a config decides the
    implementation.  for functions that the impl cannot be decided, such as
    `eglCreateImageKHR`, all implementations are called
* app calls `eglFooBar` in `/system/lib/libEGL.so`
  * `eglFooBar` calls `cnx->egl.eglFooBar`
* app calls `glFooBar` in `/system/lib/libGLESv2.so`
  * `glFooBar` calls `getGlThreadSpecific()->gl.glFooBar`
  * `glEGLImageTargetTexture2DOES` is special in that it calls
    `egl_get_image_for_current_context` to convert `EGLImage` before calling the
    hook.
* the build system defines `BOARD_EGL_CFG` for the distributed `egl.cfg`
* `libagl` supports
  * `EGL_KHR_image_base`
  * `EGL_ANDROID_image_native_buffer`
    * `eglCreateImageKHR` with `EGL_NATIVE_BUFFER_ANDROID`, which is an
      `android_native_buffer_t`.
  * `EGL_ANDROID_swap_rectangle`
  * `EGL_ANDROID_get_render_buffer`
    * return the backing `android_native_buffer_t` of an `EGLSurface`

## Buffer Management (froyo)

* buffer allocation is done by SurfaceFlinger
* buffer allocation is requested by WindowManagerService
* an app asks WindowManagerService to ask SurfaceFlinger to allocate a buffer
* SurfaceFlinger creates layers
  * a layer has two `GraphicBuffer` as can be seen in `Layer::setBuffers`
  * a `GraphicBuffer` is allocated from gralloc module or `sw_gralloc_handle_t`
  * a layer also has a `SurfaceLayer` to accept requests from
    WindowManagerService
* WindowManagerService views a `Layer` as a `Surface`
  * it requests SurfaceFlinger to create a `Layer` through
    `ISurfaceFlingerClient::createSurface`.  The returned `ISurface` is wrapped
    in a local `SurfaceControl`, as thus a `Surface`
  * it calls `ISurface::requestBuffer` to get `GraphicBuffer`
    * A `GraphicBuffer` is a `Flattenable`.  It can be flatten and unflatten
      so that it can travel as a `Parcel`
* An app asks WindowManagerService to creat a java `Surface`
  * see `android_view_Surface.cpp`
* A `Surface` inherits `android_native_window_t`
  * it can be passed to `eglCreateWindowSurface`

## buffer usages

* A `Surface` is by default `GRALLOC_USAGE_HW_RENDER`
  * it can be set wit `setUsage`
  * when a surface is locked for CPU access,
    * `GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN` is set
* libagl
  * set sw read/write often
* surfaceflinger layer
  * force the mode to sw read/write in secure mode
  * hw_texture is usually set for eglimage texturing
  * LayerDim, on msm7k, creates a sw write and hw texture buffer

## Native Types

* `eglCreateWindowSurface` takes a window of type `NativeWindowType`.  In
  android, a native window is a `Surface`.  Thus, a `Surface` wrapped in
  `EGLNativeWindowSurface` is passed to `eglCreateWindowSurface`.
* Note that framebuffer device is wrapped in `EGLDisplaySurface` and passed to
  `eglCreateWindowSurface`.
* Both `EGLDisplaySurface` and `EGLNativeWindowSurface` is a subclass of
  `EGLNativeWindowSurface` which is a subclass of `egl_native_window_t`, _THE_
  native window `GL` implementation expects.
* Similarly, a native pixmap is a `SkBitmap`.  It is wrapped in
  `egl_native_pixmap_t` and passed to `eglCreatePixmapSurface`.
* The implementation wraps all types of surfaces in a `GGLSurface`.  It is the
  natural type used by the impl. and is the object created when
  `eglCreatePbufferSurface` is called,

## `EGLNativeWindowSurface`

* inherits `EGLNativeSurface`, which inherits `egl_native_window_t`, which could
  be passed to `eglCreateWindowSurface`.
* is used by applications
* is ctor with a Surface
* as an `egl_native_window_t`,
  * `fd` is 0
  * `connect` means Surface->lock
  * `disconnect` means Surface->unlock
  * `swapBuffers` means Surface->unlockAndPost then Surface->lock
* comparing to `EGLDisplaySurface`, whose
  * `fd` is the framebuffer device
  * `connect` and `disconnect` is not set
  * `swapBuffers` copies back buffer to front buffer when no `PAGE_FLIP`; Or,
    simply flips becase `copyFrontToBack` has done the copy.

## `GLSurfaceView`

* When `start`, `EGLDisplay` is got and `eglCreateContext` is called.  It is
  done once because `eglCreateContext` is heavy/expensive.
* When a `SurfaceHolder` is got/changed, `eglCreateWindowSurface` is called on
  the surface to create `EGLSurface` and `eglMakeCurrent` is called.
* `Renderer` draws on the `EGLSurface` repeatedly in a loop.  Every time
  `onDrawFrame` returns, `eglSwapBuffers` is called.
* In view of an `egl_native_window_t`, the surface is locked most of the time
  because `connect` is called in `eglCreateWindowSurface`.

## libagl.so and pixelflinger

* pixelflinger defines GGL
* `EGLDisplay` is `egl_display_t` in libagl
* `EGLSurface` is `egl_surface_t` in libagl, which is inherited by
  `egl_window_surface_t`, `egl_pixmap_surface_t`, and `egl_pbuffer_surface_t`
* `EGLContext` is `ogles_context_t` in libagl, whose first member "rasterizer"
  is a `context_t`, whose first member "procs" is a `GGLContext`
* when init `ogles_context_t`, `ggl_init_context(&context->rasterizer)` and
  `rasterizer->base` is set to `egl_context_t`
* some GL calls modify the states of `ogles_context_t`; some others draw
  primitives through `context->prims`, which in turn calls `context->rasterizer`
* rasterizer (`context_t`) has `rast->procs (GGLContext)`, the operations, and
  `rast->state (state_t)`, the states and dst buffers
* e.g. when drawing a point,

        c->prims.renderPoint -> primitive_point -> c->rasterizer.procs.pointx ->
                --trap.c(trapezoid)--> pointx_validate -> pointx -> recti ->
                (context_t) rast->init_y, rast->rect -> scanline.c which draws
                on `context_t->state.buffers`
* e.g., to set `context_t->state.buffers`, call, `context_t->procs.colorBuffer`
* `eglMakeCurrent` calls `egl_surface_t`'s `bindDrawSurface`. It is implemented by
  calling rasterizer's colorBuffer operation.  the dst GGLSurface might or might not
  own the buffer (pbuffer owns buffer, other don't)

## `libGLESv1_CM.so`

* Provides GL entries.
* Any GL method goes through `getGlThreadSpecific` to find current `gl_hooks_t`.
  It might use implementation from `libhgl.so`, `libagl.so`, no context, etc,
  depending on current context.  When `GL_LOGGER` is defined, a call to
  `log_XXXX` is also made to log a message.
* A `gl_hooks_t` provides entries to GL functions, EGL functions, and
  extensions.

## `libEGL.so`

* The names of GL and EGL methods are built and stored in `gl_names` and
  `egl_names`.
* `libagl.so` and `libhgl.so` are represented by `egl_connection_t`.
* `load_driver` `dlopen`s the specified library.  It loops through every
  `gl_names` and `egl_names` to resolve the symbols and stores them in the
  specified `gl_hooks_t`.
* driver's `EGLContext` and `EGLSurface` are wrapped in `egl_context_t` and
  `egl_surface_t`.  They are in turn returned to users as `EGLContext` and
  `EGLSurface`.
* `eglGetDisplay` gets both sw and hw displays.
* `eglInitialize` initializes both sw and hw displays.
* `eglChooseConfig` chooses configs for both sw and hw displays.
* `eglCreateWindowSurface` and `eglCreateContext` decides which display to use.

## `GPUHardware`

* `request` makes the calling pid the owner of the GPU
  * If the GPU is owned for the first time, two `GPUAreaHeap` is created for
    `/dev/pmem_gpu?`.  A `GPURegisterHeap` is also created.
  * A client and the `mCurrentAllocator` is created.
  * If `request` is called with a `IGPUCallback` and a
    `ISurfaceComposer::gpu_info_t`, callback is also linked to death and heap
    infos are returned (to client for DRI!)
* `revoke` is called through `revokeNotification` when the client does not use gpu anymore
  * it signals mCondition
  * it calls `releaseLocked` to revoke current client's heaps
* if B tries to own gpu while A is the owner
  * `takeBackGPULocked` is called to notify A `gpuLost` and waits on mCondition
  * A calls `revoke`
  * `releaseLocked` is called to make sure A is not the owner anymore
* `SurfaceFlinger` calls `friendlyRevoke` when console released (chvt to another
  one) or remote calls `revokeGPU`.
* Comparison
  * `request(pid)` makes the pid current owner and returns the dealer
  * `request(...)` makes the pid current owner, registers the callback, and
    returns `gpu_info_t`
  * `revoke(pid)` (current owner) acks the revoke and releases the gpu
  * `friendlyRevoke(void)` notifies current owner and waits before releasing
     the gpu
  * `unconditionalRevoke(void)` releases the gpu

## Observations

* For `Surface`, an `EGLSurface` locks the surface until destroied.  When
  `swapBuffers`, `unlockAndPost` is immediately followed by `lock`.
  `bindDrawSurface` is then called to update buffer info (offset change caused
  by flipping).
* 

## DRI and GEM

* When a `Surface` is created with memory type `GPU`, the two IMemoryHeaps are
  based on `GEM`.  Maybe MemoryHeapGem?  See MemoryHeapPmem and MemoryHeapBase.
* Per client heap is returned by SurfaceHeapManager, which returns a dealer from
  GPU when requested.  See `GPUHardware::request`.
* Should be able to ask heap for GEM name.  Extend IMemory? 
* Lock surface and pass back buffer's GEM name to eagle
* After eglSwapBuffers, eagle is done.  unlockAndPost!
* EGLDisplaySurface uses modesetting
  * display has two scan-out buffers, and gl buffers (front, back, ...)
  * all gl drawings will finally make to front buffer
  * swapbuffer copies front buffer to scan-out buffer
* DisplayHardware manages opengl for surfaceflinger
  * it helps create display, context, and surface
  * it hides egl
  * hw.flip -> eglSwapBuffers -> display.swapBuffers
* note that GGLSurface draws on buffer
* At this point, libagl draws on gem buffers
* use gem in Surface for preparation
* eagle-based libhgl.  EGLDisplaySurface and gem-backed EGLNativeWindowSurface
  both can be accelerated
* GEM is a MemoryDealer

## DRI and GEM impl.

* class MemoryHeapGem : public class MemoryHeapBase
  is a gem buffer
  initialized with a device
  getHeapID is gem name
  getHeapHandle is gem handle (NEW)
* class GEMHardware: public class GPUHardwareInterface
  has a internal class GemDealer : public class MemoryDealer
  Whenever a dealer is requested, return a new GemDealer based on a
  newly allocated MemoryHeapGem of size 2M.
* When SurfaceHeapManager is asked for a GPU based dealer, a GemDealer is
  returned
* EGLDisplaySurface for modesetting
  DisplayHardware should reprogram crtc when acquiring screen

## eagle and Buffers

* In the best case, eagle should draw to the back buffer of the native window.
  `eglSwapBuffers` switch front/back buffers, and dri should switch too.  The
  buffer preserve bit is not set.
* It is not possible with dri driver.
* For window surface, eagle should draw to a buffer of EGL's own.
* A window surface should lock NativeWindow.  `eglSwapBuffers` should copy color
  buffer to NativeWindow's back buffer, unlockAndPost, and lock again.  The
  color buffer is left front and there should be no left back.  The buffer
  preserve bit is set.
* For EGLKMSSurface, we have total control.  EGLKMSSurface should be
  single-buffered due to the above.
* `eglCopyBuffer` copies the color buffer to a native pixmap.  Might consider
  use it.
* copies of EGLKMSSurface
  * `copyFrontToImage` copies front buffer to image
  * `copyBackToImage` copies back buffer to image
  * `copyFrontToBack` is used by `DisplayHardware` when the display surface is
    `BUFFER_PRESERVED`.  It copies clean region from front buffer to back
    buffer.  When `BUFFER_PRESERVED` is not set, the dirty region is always set
    to hw size (or `mInvalidRegion`) so that the whole scene is redrawn.  When
    set, only the dirty region is redrawn.  The rest is copied from front to
    back.
  * It should have `BUFFER_PRESERVED` set and its `copyFrontToBack` should not
    do anything.  Its `SwapBuffers` should copy everything from back to front,
    obeying swap region.
* When a surface is locked, it copies front to back to keep both buffers in sync
  (`eBufferDirty` is set only when a surface is just initialized).  The copied
  region is previous dirty minus new dirty.  Since `EGLNativeWindowSurface`
  always lock without a new dirty, the new dirty is always set to whole surface.
  That is, there is no copy from front to back for `EGLNativeWindowSurface`.
* `SwapBuffers` of Surface `unlockAndPost` and `lock` again.  Its
  `mSwapRectangle` is always sent with `unlockAndPost` as dirty region if it is
  valid.  (It is usually invalid.)
* In summary, EGL will own a buffer for any kind of native window.  It is
  OpenGL's front left buffer.  EGLKMSSurface will own a buffer and is updated
  when swap buffers.  EGLNativeWindowSurface will own two buffers and its back
  buffer is updated, posted, and switched with the front buffer.  The updating
  always redraw everything, thus the switching will not cause copy front to
  back.
* `UPDATE_ON_DEMAND` is always set.

## eagle and DRI auth

* `SurfaceFlinger` is modified to allow DRI auth
* requires root priv? or modify kernel?
* When or who to make the function call?
* 

## eagle and Surface GEM

* `GEMHardware` to provide `MemoryGem`
* `Surface` with GEM support
* `EGLNativeWindowSurface` sets GEM names
* When the back buffer is posted, `Layer::reloadTexture` is called.  It passes
  the `GGLSurface` to `LayerBase::loadTexture`.  `DIRECT_TEXTURE` allows the
  buffer to be used directly, without making a copy.  However, see
  <http://www.mail-archive.com/android-porting@googlegroups.com/msg03302.html>
  It is almost not used anymore.
* New code relies on `copyblt` instead.  That is, texture are uploaded normally
  but probably not used.  See `Layer::onDraw`.
* post-cupcake, `EGL_KHR_image_base` is used.
* what do we do?  `eglBindTexImage`?
* resize?

## eagle and OpenGL ES

* No fixed point support
* No drawTexture
* No thread-safety, no boot animation
* should be solved by switching to gallium

## Java bindings

* `javax.microedition.khronos.{egl,opengles}` describes one binding for EGL and
  GLES called JSR239
* `com.google.android.gles_jni` is an implementation of JSR239 using JNI
* `android.opengl` defines another Android-specific GLES binding, implemented
  also with JNI, as well as high-level constructs
  * all functions are static and are said to be faster
  * see commit 27f8002e591b5c579f75b2580183b5d1c4219cd4
