Mesa and Its Driver
===================

## mesa/drivers/ and mesa/

- call flow: glXXX call -> glapi -> mesa -> driver
- impl flow: driver -> mesa -> glapi
- reset of mesa/ is like pixelflinger in android
- important types

        typedef struct __GLcontextRec GLcontext;
        typedef struct __GLcontextModesRec GLvisual;
        typedef struct gl_framebuffer GLframebuffer;
        struct gl_renderbuffer
- ctx is huge.  It has (Save, Exec, and CurrentDispatch), Visual, (DrawBuffer, ReadBuffer), (Driver, DriverCtx), module contexts
- ctx is initialized by `_mesa_initialize_context`.  Exec and Save points to `alloc_dispatch_table`.
  - Exec is initialized with `_mesa_init_exec_table`.
  - _mesa_loopback_init_api_table: there are 28 glColor{3,4}{bdis ui us ub}[v], 27 of them are loopbacked to the most generic one.
  - _mesa_init_exec_vtxfmt: vertex format (glColor, etc.) dispatch are done lazily when used.  This is a large set of functions,
				which are intalled/restored with different TNL.  Lazy dispatch improves install/restore.
				The real functions are at ctx->TnlModule.Current.
- ctx->swtnl_im is initialized by `_vbo_CreateContext`; deep in
  `_vbo_CreateContext`, real vertex format functions is installed,
  using impl from `vbo_exec_vtxfmt_init`
- ctx->swtnl_context is initialized by `_tnl_CreateContext`
- renderbuffer ops are defined usually with the help of swrast/s_spantemp.h

## mesa/drivers/fbdev/ as an example

- usage: glFBDevCreateVisual, glFBDevCreateBuffer, glFBDevCreateContext, glFBDevMakeCurrent, glFBDevSwapBuffers
- all calls into mesa/
- glFBDevCreateVisual: allocate a (child of) GLvisual based on fb_fix_screeninfo and fb_var_screeninfo
			keyword: `_mesa_initialize_visual`
- glFBDevCreateBuffer: allocate a (child of) GLframebuffer using provided GLvisual and (mmapped) fb0
			keyword: `_mesa_initialize_framebuffer,
				swrast/s_spantemp.h, _mesa_init_renderbuffer, _mesa_add_renderbuffer
				_mesa_add_soft_renderbuffers`
- glFBDevCreateContext: allocate a (child of) GLcontext using based on provided GLvisual
			keyword: `_mesa_init_driver_functions, _mesa_initialize_context
				lots other context init functions`

## mesa/drivers/dri/

- three DRIs: DRI (legacy), DRI2, DRISW
  loader should provide: `__DRI_GET_DRAWABLE_INFO`, `__DRI_DRI2_LOADER`, `__DRI_SWRAST_LOADER` respectively
- important types in dri_interface.h, public but opaqueue

        typedef struct __DRIscreenRec		__DRIscreen;
        typedef struct __DRIcontextRec	__DRIcontext; // inherits drm_context_t for DRI(2), GLcontext for DRISW
        typedef struct __DRIdrawableRec	__DRIdrawable; // inherits drm_drawable_t for DRI(2), GLframebuffer for DRISW
        typedef struct __DRIconfigRec		__DRIconfig; // inherits __GLcontextModes
        typedef struct __DRIframebufferRec	__DRIframebuffer;
        typedef struct __DRIversionRec	__DRIversion;

## mesa/drivers/dri/swrast/ as an example

- __DRI_SWRAST provides createNewScreen and createNewDrawable which are what
  `__DRI_CORE` lack.
- driCreateNewScreen invokes driCreateConfigs to create driver configs
- driCreateNewDrawable quite normal.  It is simpler than dri legacy's, just like dri2. swrast_renderbuffer inherits gl_renderbuffer
- driCreateNewContext quite normal
- driSwapBuffers putImage backbuffer if double-buffered.  buffers are rendered through swrast_span
- driBindContext quite normal except swrast_check_and_update_window_size

## mesa/drivers/dri/common/dri_util.c

- the common interface for DRI drivers, with `__driDriverExtensions` as the entry point
- provides many extension helpers which DRI drivers could optionally support
- in binding context, DRI1 calls getDrawableInfo to get info like x, y, w, h, cliprect
- if loader supports damage (dri1?), reportDamage after swap buffer
- pixmap drawable is never created, and never supported
- dri2 drawable has one cliprect and one back cliprect of drawable size
- dri1 creates new screen with drmMapped sarea and fb
- util.c gl extensions and dri driver extensions


