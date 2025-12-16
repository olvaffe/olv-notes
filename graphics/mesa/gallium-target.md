Mesa Gallium
============

## gallium

- directories
  - `GALLIUM_AUXILIARY_DIRS`: subdirs of `auxiliary` to build
  - `GALLIUM_DRIVERS_DIRS`: subdirs of `drivers` to build
  - `GALLIUM_STATE_TRACKERS_DIRS`: subdirs of `state_trackers` and
    `winsys/drm/intel` to build
  - `GALLIUM_WINSYS_DIRS`: subdirs of `winsys` to build.
    - `GALLIUM_WINSYS_DRM_DIRS`: subdirs of `winsys/drm/` to build.
  - By default (`configs/default`),

    SRC_DIRS = mesa gallium egl gallium/winsys glu glut/glx glew glw
    DRIVER_DIRS = x11 osmesa
    GALLIUM_DIRS = auxiliary drivers state_trackers
    GALLIUM_AUXILIARY_DIRS = draw translate cso_cache pipebuffer tgsi sct rtasm util indices
    GALLIUM_DRIVERS_DIRS = softpipe i915simple failover trace
    GALLIUM_WINSYS_DIRS = xlib egl_xlib
    GALLIUM_STATE_TRACKERS_DIRS = glx
  - `config/linux-dri`

    SRC_DIRS := glx/x11 egl $(SRC_DIRS)
    DRIVER_DIRS = dri
    WINDOW_SYSTEM = dri
    GALLIUM_WINSYS_DIRS = drm
    GALLIUM_WINSYS_DRM_DIRS = intel
    GALLIUM_STATE_TRACKERS_DIRS = egl
- `src/mesa/`
  - provides `libmesagallium.a`, `libglapi.a`, the mesa opengl state tracker
- `src/gallium/`
  - `auxiliary` provides a `.a` in each subdir
  - `drivers` provides a `.a` in each subdir; they are pipe drivers (or pipes).
  - The above two are self-contained and are supposed to be used on a wide range
    of platforms.
  - `state_trackers`
    - `dri` provides `libdridrm.a`, DRI driver interface used by `drm` winsys
    - `egl` provides `libegldrm.a`, EGL driver interface used by `drm` winsys
    - `glx` provides `libxlib.a`, GLX API over core X protocol
    - `python` provides python binding
    - `vega` provides `libOpenVG.so`, linking to many auxiliaries
  - `winsys`
    - `xlib` provides `libGL.so`, linking to all pipes, all auxiliaries,
      `libxlib.a`, `libglapi.a`, and `libmesagallium.a`.  It uses mesagallium st
      and any pipe available.
    - `egl_xlib` provides `egl_softpipe.so`, linking to all pipes and all
      auxiliaries.  It is an EGL driver used by `src/egl/`.  Only softpipe is
      used.  Any st (like `libOpenVG.so`) can be used with it.
    - `drm` provides `xxx_dri.so` or `EGL_xxx.so`.  That is, it provides either
      EGL driver or DRI driver.  All auxiliaries and `libmesagallium.a` are
      always linked.
- `src/gallium/winsys/drm/intel`
  - `gem` is always built, and provides `libinteldrm.a`.  It supports softpipe
    and i915simple.
  - `egl` provides `EGL_i915.so`, linking to `libegldrm.a`, `libinteldrm.a`,
    `libsoftpipe.a`, and `libi915simple.a`.
  - `dri` provides `i915_dri.so`, linking to `libdridrm.a`, `libinteldrm.a`,
    `libsoftpipe.a`, and `libi915simple.a`.
- winsys combines pipe driver(s) and talks to `state_tracker`.
  e.g. `egl_softpipe.so` links to `libsoftpipe.a` and calls to functions like
  `st_create_context`, provided by, say, `libOpenVG.so` state tracker.
  Or, `i915_dri.so` links to `libi915simple.a` and `libdridrm.a`, which talks to
  a state tracker, say `libmesagallium.a`.
- winsys would call certain pipe and st functions.  What a winsys calls might
  not be defined by every pipe and st.  Thus, one cannot put random winsys,
  pipe, and st together.

## Impl.

- Initialization
  - winsys should create a `struct pipe_winsys`, which is used for buffer
    creation.
  - winsys should provide a list of supported configs.
  - `struct pipe_winsys` is passed to desired pipe driver to create
    `struct pipe_screen`.  Note that the pipe driver is known at this point.
- Context
  - pipe driver is asked to create a `struct pipe_context`.
  - pipe context is then passed to state tracker to `st_create_context` a
    `struct st_context`.  Which state tracker is used depends on which client
    api is linked.
- Framebuffer
  - state tracker is asked to to `st_create_framebuffer` a
    `struct st_framebuffer`.  It is only a record.  There is no real buffer
    associated with it.  It has no context either.
- `st_make_current` is called to make context/fb current.
- To swap buffers, a fb is `st_get_framebuffer_surface` to return a renderbuffer
  (called `struct pipe_surface`).  The contents is copied to the native window.

## EGL drivers

- `EGL_i915.so` and `egl_softpipe.so`
- state trackers defining
  `st_api_OpenGL`/`st_api_OpenGL_ES1`/`st_api_OpenGL_ES2`/`st_api_OpenVG` will
  have respective `EGL_<CLIENT-API>_BIT` set when used with `libEGL.so` and
  `egl_softpipe.so`.

## OpenGL ES

- `gallium/state_trackers/es/` provides `libGLESv1_CM.so` and `libGLESv2.so`
- It soft links to most of mesa's source files and builds `libmesa.a` under its
  `mesa` directory.  Some files are omitted as they are not needed.  Some are
  specific to ES (`state_tracker/st_cb_drawtex.c`), and some (`main/pixel.c` and
  `main/mfeatures.h`) are replaced.
- All libs and objects under `es1` are used to create `libGLESv1_CM.so`.
- `es2` is similar.
- In OpenGL, what a function does might depend on the context.  Thus, a dispatch
  table is used.  It is not the case with OpenGL ES.  A no-op replacement can be
  found under `es-common/st_glapi.c`.
- How a `glXXX` is implemented usually has a rule and is generated.  Some are
  not.  For those cannot be generated, they are listed in `es1_special` and
  `es2_special`.
- Three files are generated under either `es1` or `es2`
  - `st_es1_generated.c` generated by `es_generator.py` and `APIspec.txt`
  - `st_es1_getproc_gen.c` generated by `es_getproc_gen.py` and `APIspec.txt`
  - `st_es1_get.c` generated by `get_gen.py`
  - Similar to `es2`
- `st_es1_getproc_gen.c` provides `_glapi_get_proc_address` to map function
  names to function pointers.  According to EGL spec, only those that are
  extensions are mapped.  It is used to implement `st_get_proc_address`,
  `eglGetProcAddress` or `glXGetProcAddressARB`.
- `st_es1_get.c` provides `_mesa_GetBooleanv`, `_mesa_GetFloatv`, and
  `_mesa_GetIntegerv`.  They are used to implement `st_GetTYPEv` and
  `glGetTYPEv`.
- `st_es1_generated.c` provides OpenGL ES API.  It does not use dispatch table
  and the generated code is very complex.  The generator allows error checking
  the inputs and allows calling arbitrary "alias" (e.g. `glActiveTexture` calls
  `_mesa_ActiveTextureARB`).
