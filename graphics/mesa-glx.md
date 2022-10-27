Mesa and GLX
============

## Spec

* In xserver, a visual describes the pixel format.  Different screen and
  different depth support different visuals.
  * `XVisualInfo`, on the other hand, is the tuplet of
    `(screen, depth, visual)`.

## xserver/hw/xfree86/dixmods/glxmodule.c:
* dlsym glxModuleData for setup function
* GlxPushProvider `__glXDRISWRastProvider`, `__glXDRIProvider`,
  `__glXDRI2Provider`
* LoadExtension, calling GlxExtensionInit

## xserver/glx/
* each provider screenProbe all screens
* register ext., with dispatch `__glXDispatch`
* `__GLX(DRI)screen` is associated with Screen
* `CALL_xxx` -> where is xxx defined?


## xserver/hw/xfree86/dri2/

## GLX

* glx/x11 provides libGL.so which implements GLX using DRI/DRI2.
* mesa/drivers/x11 provides libGL.so which implements GLX using core X11 protocol

## `GLX_EXT_texture_from_pixmap` and `glXBindTexImageEXT`

* if a xxx_dri.so is dri2 and provides `__DRI_TEX_BUFFER`,
  `__glXEnableDirectExtension` is called to enable the extension.
* glXQueryExtension reports support

## mesa/glx/x11/

* important GLX types

        typedef struct __GLXscreenConfigsRec __GLXscreenConfigs;
        typedef struct __GLXcontextRec __GLXcontext;
        typedef struct __GLXdrawableRec __GLXdrawable;
        typedef struct __GLXdisplayPrivateRec __GLXdisplayPrivate;
* important GLXDRI types

        typedef struct __GLXDRIdisplayRec __GLXDRIdisplay;
        typedef struct __GLXDRIscreenRec __GLXDRIscreen;
        typedef struct __GLXDRIdrawableRec __GLXDRIdrawable;
        typedef struct __GLXDRIcontextRec __GLXDRIcontext;
        typedef struct __GLXDRIconfigPrivateRec __GLXDRIconfigPrivate; // inherits __GLcontextModes
* important per-protocol (DRI, DRI2, DRISW) GLXDRI types, inheriting non-private ones

        typedef struct __GLXDRIdisplayPrivateRec __GLXDRIdisplayPrivate;
        typedef struct __GLXDRIcontextPrivateRec __GLXDRIcontextPrivate;
        typedef struct __GLXDRIdrawablePrivateRec __GLXDRIdrawablePrivate;
* import GLX protocol-defined types

        typedef XID GLXPixmap;
        typedef XID GLXDrawable;
        /* GLX 1.3 and later */
        typedef struct __GLXFBConfigRec *GLXFBConfig;
        typedef XID GLXFBConfigID;
        typedef XID GLXContextID;
        typedef XID GLXWindow;
        typedef XID GLXPbuffer;
* some of the driXxxXxx functions talk to kernel; others talk to X

## GLX Howto

* There are 4 functions `glXChooseVisual`, `glXCreateContext`, `glXMakeCurrent`,
  and `glXSwapBuffers` that apps usually call.
* Take a look at `__glXInitialize` first
  * It queries the server for GLX extension.
  * It allocates `struct __GLXdisplayPrivate` to remember how to talk to the
    server in GLX.  It also calls `dri2CreateDisplay`, `driCreateDisplay`, and
    `driswCreateDisplay` to prepare for DRI.
  * It calls `AllocAndFetchScreenConfigs` to allocate configs.  For each screen,
    it calls one of the DRI's `createScreen` to get a single `driScreen`.  It
    sends `X_GLXGetVisualConfigs` and `X_GLXGetFBConfigs` to get per-screen
    `__GLcontextModes` which are stored in `visuals` and `configs` respectively.
    The latter is only available for GLX 1.3 and newer.
* Comparing visuals and fbconfigs,
  * FB configs are OpenGL's idea.  Visuals are X's idea.  In `__glXScreenInit`
    of X, each X visual is examined.  If there is a compatible fb config, the
    config is stored in `pGlxScreen->visuals`, as a glx visual.  Conversely, for
    fb configs that are compatible with X, it is added as a visual.
  * `glXChooseVisual` returns visuals while `glXChooseFBConfig` returns
    fbconfigs
  * `glXCreateContext` takes visual while `glXCreateNewContext` takes fbconfig.
  * `XCreateWindow` takes visual while `glXCreateWindow` takes fbconfig (and a
    window).
  * The latter enables `glXCreatePbuffer`.
* 

## mesa/glx/x11/drisw_glx.c as an example
* when loader driCreateScreen, load swrast_dri.so and looks for `__DRI_DRIVER_EXTENSIONS`.
* loader make sure `__DRI_CORE` and `__DRI_SWRAST` are supported
* loader uses `__DRI_SWRAST`'s createNew{Screen,Drawable}

## mesa/glx/x11/dri2_glx.c as an example

* dri2CreateDisplay querys for DRI2 extension and version.  Installs
  dri2CreateScreen and dri2DestroyDisplay
* dri2CreateScreen connects to DRI2 to get the driver and device names.
  Suitable xxx_dri.so is dlopen()ed and `__DRI_CORE` and `__DRI_DRI2` are
  looked up.  /dev/dri/cardX is opened.  Authenticate!  A `__DRIscreen` is
  created using the driver.  psc->{visuals,configs} is filtered.
  Installs functins into psp.  More extensions are queried through core->getExtensions.
* auth goes through: client creates a magic, send the magic to x server,
  server auths the magic.  client is then called authenticated.
* dri2CreateContext creates DRI2 context through psc->dri2->createNewContext.
* dri2CreateDrawable notifies the server about a new DRI2Drawable and psc->dri2->createNewDrawable
* dri2BindContext calls core->bindContext
* dri2GetBuffers is a loader extension.  It is called when gem is needed by xxx_dri.so.  It calls
  DRI2GetBuffers to ask the x server.
* dri2SwapBuffers copies what the client draw on the gem to server through
  XFixesRegion and DRI2CopyRegion.  (The gem is usually offscreen for
  redirected direct rendering? locking?)
* GetBuffers and CopyRegion requests are at last handled by the server video
  driver, which hooks up the callbacks by calling DRI2ScreenInit.
  GetBuffers involves DRM_IOCTL_I915_GEM_CREATE
  CopyRegion involves gc->CopyArea from back buffer to drawable

## example

* a client create a window, DRI (glXCreateWindow), and swap buffer
* glXCreateWindow -> ask server for a glx drawable (GLXdrawable) and then
  dri2CreateDrawable -> ask server for a dri2 drawable (__GLXDRIdrawable) and
  then xxx_dri.so createNewDrawable -> creates a `__DRIdrawable` -> BindContext
  and start drawing on buffers from GetBuffers
* glXSwapBuffers -> dri2SwapBuffers -> asks server to CopyRegion from back left
  to front left.
* on the server side ...
* GetBuffers uses window's pixmap for front left, use newly created pixmaps for some others
  This can be seen in intel driver, where pixmaps are associated with a bo.
  Window's pixmap is screen's pixmap, which points to the mmap()ed framebuffer.
  Deep in I830ScreenInit, it can be noted that DRI2 requires UXA.
* CopyRegion copies src to drawable by gc->ops->CopyArea.
* in intel dri2, all pixmaps are backed by bo, thus all offscreen in exa terms.

## Frame Throttling

- many apps have mainloops that keep
  - `glCear`
  - draw frame
  - `glXSwapBuffers`
- they expect GL/GLX to throttle
- `glCear`
  - in mesa, this calls `st_validate_state` which calls down into
    `st_framebuffer_validate` and `loader_dri3_get_buffers`
  - when all buffers in the swapchain are in use, `loader_dri3_get_buffers` is
    blocked inside `dri3_find_back` until one of the buffer becomes idle
- draw frame draws to the acquired buffer
- `glXSwapBuffers`
  - in mesa, this calls `dri3_swap_buffers` and `loader_dri3_swap_buffers_msc`
  - `loader_dri3_swap_buffers_msc` performs an implicit flush
    - `loader_dri3_flush` calls `dri_flush`
    - `dri_flush` calls `st_thread_finish` and `st_flush`
- when is a buffer consider idle in `dri3_find_back`?
  - vblank wakes up wayland
    - wayland latches all client buffers and composites them
    - it also knows the previous scanout buffer(s) are idle and notifies
      xwayland
  - xwayland notifies mesa and `dri3_find_back` returns
  - when if crtc is off?
    - xwayland `present_fake_screen_init` fakes 1fps

## two threads `glXMakeCurrent` and `glXSwapBuffers`

* Have a look at `glXGetCurrentContext` for `__GLXcontext` first
  * With `GLX_USE_TLS`, `__glX_tls_Context` is TLS.
  * With `PTHREADS`, it is stored under the key `ContextTSD`.
  * Without thread support, it is a global `__glXcurrentContext`.
* In DRI, every thread has and uses its context.
* In indirect rendering,
  * X server calls `AddExtension` to dispatch GLX through `__glXDispatch`.
  * It calls `__glXleaveServer` and `__glXenterServer` before and after
    dispatching.
  * GL functions are dispatched to `__glXDisp_Render`.  It calls
    `__glXForceCurrent` to make client's context current before rendering.
  * `glXMakeCurrent` also makes a client's context current.  Since X is
    single-threaded, there is no more than one context current at any time.

## `Visual` and `GLXFBConfig`

* A `Screen` has visuals.  A `__GLXscreen` has fbconfigs.
* fbconfigs of GLX screen are converted to visuals
  * For each normal visual, a best config is picked and associated with it.
  * For each config that has not been converted, a new visual is created.
  * See `__glXScreenInit`.

## `Drawable` and `GLXDrawable`

* `glXCreateWindow` and `glXCreatePixmap` create `GLXWindow` and `GLXPixmap`
  from an existing `Window` and `Pixmap`.
  * X server uses DRI exclusively, and `GLXWindow` and `GLXPixmap` both
    correspond to `__DRIdrawable`s.
  * DRI draws using the format of the given `GLXFBConfig`.  If it is not
    compatible with the visuals of `Window` and `Pixmap`, bad things happen.
  * In `glXCreateWindow`, if `Window` is not created with the visual the same as
    one from `glXGetVisualFromFBConfig`, it is `BadMatch`.  See 3.3.4.
  * In `glXCreatePixmap`, the `Pixmap` must be created with a depth determined
    by the `GLXFBConfig`.  That is, the depth of the visual of the config must
    be the same as the depth of the pixmap.
* `glXMakeCurrent` allows `Window` (but not `Pixmap`) to be made current.
  * See section 2.1.  a `GLXDrawable` is the union of
    `{ GLXWindow, GLXPixmap, GLXPbuffer, Window }`.
  * The corresponding `GLXWindow` is allocated automatically, using the config
    of the to-be-current context.  The config and the visual of the `Window` is
    checked for compatibility.  See `__glXGetDrawable`.

## `EXT_texture_from_pixmap`

* Given a `Window`, compiz calls `XCompositeNameWindowPixmap` on the `Window` to
  get the offscreen `Pixmap`.  `glXCreatePixmap` is called on the `Pixmap` to
  get the `GLXPixmap`.  `glXBindTexImageEXT` is called on the `GLXPixmap` to
  bind it as a texture.
* When `Pixmap` is updated, how is it reflected in `GLXPixmap`?
  * 
* A thought about `EGLImage`.  For a client api, `EGLImage` is
  `struct _EGLImage { struct pipe_surface *surface; };`.  The pipe surface is
  filled by an EGL driver.  Given a `Pixmap`, suppose it has an associated
  `GLXPixmap` and we have access to it.  We can ask glx to return the associated
  `__DRIdrawable`.  It has a dri buffer associated, which is created by
  `dri_create_buffer` when gallium dri driver is used.  The dri buffer is
  `struct dri_drawable`.  It has a `struct st_framebuffer`, and
  `st_get_framebuffer_surface` can be called to reutnr the pipe buffer.
* A way to let EGL driver get the pipe surface of a `Pixmap`.   How?
  * In GLX, glx must be avaible to find `GLXPixmap` from a `Pixmap`.  And glx
    must be able to ask the dri driver to return the pipe surface of a
    `__DRIdrawable`, if it is gallium based.

## Drawable invalidation

* When GLX detects drawable change, it calls `__DRI2_FLUSH`'s `invalidate` to
  notify the driver
* The driver calls `dri2InvalidateDrawable` to update the drawble's stamp,
  `drawable->dri2.stamp`
* For a `__DRIdrawable`,
  * `lastStamp` is the local stamp, initially 0
  * `dri2.stamp` is the server stamp, initially `lastStamp + 1`
  * `pStamp` points to the server stamp (for DRI1, not used for DRI2)
  * when `lastStamp != dri2.stamp`, the driver validates the drawable and set
    `lastStamp to dri2.stamp`
* For a `__DRIcontext`,
  * there is `dri2.draw_stamp` and `dri2.read_stamp`, the stamps of the
    drawables as the context knows
  * when `dri2.draw_stamp` is not equal to `dri2.stamp` of the draw drawable,
    the context is updated

## DRI3

 - on-screen render area is called Window in X11
 - off-screen render area is called Pixmap in X11
 - internally, they are all just Pixmap (because of compositor?)
 - for windows, mesa allocates buffers and use DRI3PixmapFromBuffer to create
   Pixmaps for presentation
 - for pixmaps, the server allocates buffers and Mesa uses
   DRI3PixmapFromBuffer to get the buffers
 - in Xwayland, a Pixmap is created by allocating `gbm_bo` or importing
   dma-bufs.  The `gbm_bo` is wrapped in a `wl_buffer`
   - in virtio, that is `DRM_IOCTL_VIRTGPU_RESOURCE_CREATE`
