Mesa and DRI
============

## DRI interface

- `dri_interface.h` defines the DRI interface
  - it should be considered internal to mesa nowadays
  - the only external users are xserver and minigbm
    - the plan is to make the legacy extensions that xserver relies stable
      ABI, and the rest is fully internal

## DRI Loaders

- egl, glx, and gbm use `loader.h` to load DRI drivers
- egl/x11
  - `eglInitialize`
  - `dri2_initialize`
  - `dri2_initialize_x11`
  - `dri2_initialize_x11_dri3`
  - `dri3_x11_connect`
    - `loader_dri3_open` queries the fd from the xserver
    - `loader_get_user_preferred_fd` distinguishes render and display fds,
      if prime is enabled
    - `loader_get_driver_for_fd` queries the driver name from the fd
  - `dri2_load_driver_dri3`
    - `loader_open_driver` loads the DRI driver and returns
      `__DRIextensionRec` array of the driver
    - `loader_bind_extensions` binds the driver extensions listed in
      `dri3_driver_extensions`
  - `dri2_create_screen`
  - `dri2_setup_extensions`
    - `loader_bind_extensions` again binds the driver extensions listed in
      `dri2_core_extensions` and `optional_core_extensions`
- egl/wayland
  - `dri2_initialize_wayland` instead of `dri2_initialize_x11`
  - `dri2_initialize_wayland_drm`
  - `dri2_initialize_wayland_drm_extensions`
    - `default_dmabuf_feedback_main_device`, if the compositor is modern
      - `loader_get_render_node` returns the render node path from `dev_t`
      - `loader_open_device` opens the render node
    - `wl_drm_bind`, if the compositor is older
      - `loader_open_device` opens the device (primary or render) node path
  - the rest is just like egl/x11
- glx
  - `__glXInitialize`
  - `AllocAndFetchScreenConfigs`
  - `dri3_create_screen`
    - `loader_dri3_open`, `loader_get_user_preferred_fd`, and
      `loader_get_driver_for_fd`, `loader_open_driver` and
      `loader_bind_extensions` are just like in egl/x11
    - `loader_bind_extensions` is called with `exts` array instead
  - `dri3_bind_extensions`
    - `loader_bind_extensions` again with another `exts` array
- gbm
  - `gbm_create_device`
  - `_gbm_create_device`
  - `find_backend`
  - `backend_create_device`
  - `dri_device_create`
  - `dri_screen_create`
    - `loader_get_driver_for_fd` just like in egl
  - `dri_screen_create_for_driver`
    - `loader_open_driver` and `loader_bind_extensions` just like in egl
    - `loader_bind_extensions` is called with `gbm_dri_device_extensions`
    - `loader_bind_extensions` is called again with `dri_core_extensions`

## DRI extensions

- driver/loader extension exchanges
  - `loader_open_driver` uses `dlsym` to get the static list of driver
    extensions, which is `galliumdrm_driver_extensions`
    - if swrast, it is `galliumsw_driver_extensions` or
      `dri_swrast_kms_driver_extensions`
  - loaders call `driCreateNewScreen2` from the DRI driver to create a screen
    - the list of loader extensions is passed to the driver
      - egl/x11 uses `dri3_image_loader_extensions`
      - egl/wayland uses `dri2_loader_extensions`
      - glx uses static `loader_extensions`
      - gbm uses static `gbm_dri_screen_extensions`
    - the list of static driver extensions is passed back to the driver
      - this is done because hw drivers and swrast drivers, which have
        different static driver extensions, live in the same DRI driver
        library
  - loaders call `driGetExtensions` from the DRI driver to get the per-screen
    list of driver extensions
    - the list is initialized by `dri2_init_screen_extensions` dynamically
- static driver extensions
  - `loader_bind_extensions` is called after `loader_open_driver`
    - the loader (GLX, GBM, or EGL) provides a list of required and optional
      extensions
    - the function matches against the list of extensions returned by the DRI
      driver
  - GLX asks for `exts`
    - `__DRI_CORE`
    - `__DRI_IMAGE_DRIVER`
    - `__DRI_MESA`
  - GBM asks for `gbm_dri_device_extensions`
    - `__DRI_CORE`
    - `__DRI_MESA`
    - `__DRI_IMAGE_DRIVER`
  - EGL asks for `dri3_driver_extensions`
    - `__DRI_CORE`
    - `__DRI_MESA`
    - `__DRI_IMAGE_DRIVER`
    - `__DRI_CONFIG_OPTIONS`
  - these functions are used
    - `__DRImesaCoreExtensionRec::queryCompatibleRenderOnlyDeviceFd`
      - when the display server gives the loader an fd whose device is
        kms-only, this returns an fd for another device that is capable of
        rendering
    - `__DRImesaCoreExtensionRec::createNewScreen` create a new screen
      - `__DRIimageDriverExtensionRec::createNewScreen2` is the older way
      - `__DRIimageDriverExtensionRec::getAPIMask` is the older way to query
        supported apis (gl, gles2, gles3, etc.)
    - `__DRIcoreExtensionRec::destroyScreen` destroys a screen
    - `__DRIcoreExtensionRec::getExtensions` returns the per-screen extensions
      - this depends on the caps of the `pipe_screen`
    - `__DRIconfigOptionsExtensionRec::getXml` is for `EGL_MESA_query_driver`
      - it is used by driconf gui app
    - `__DRIcoreExtensionRec::getConfigAttrib` queries a config attr
    - `__DRIcoreExtensionRec::indexConfigAttrib` enumerates config attrs
    - `__DRImesaCoreExtensionRec::createContext` creates a new context
      - `__DRIimageDriverExtensionRec::createContextAttribs` is the older way
    - `__DRIcoreExtensionRec::destroyContext` destroys a context
    - `__DRIcoreExtensionRec::bindContext` binds a context
    - `__DRIcoreExtensionRec::unbindContext` unbinds a context
    - `__DRIimageDriverExtensionRec::createNewDrawable` creates a new drawable
    - `__DRIcoreExtensionRec::destroyDrawable` destroys a drawable
- loader extensions
  - loader provides a list of extensions to the driver in `createNewScreen`
  - GLX provides `loader_extensions`
    - `__DRI_IMAGE_LOADER`
    - `__DRI_USE_INVALIDATE`
    - `__DRI_BACKGROUND_CALLABLE`
  - GBM provides `gbm_dri_screen_extensions`
    - `__DRI_IMAGE_LOOKUP`
    - `__DRI_USE_INVALIDATE`
    - `__DRI_DRI2_LOADER`
    - `__DRI_IMAGE_LOADER`
    - `__DRI_SWRAST_LOADER`
    - `__DRI_KOPPER_LOADER`
  - EGL/X11 provides `dri3_image_loader_extensions`
    - `__DRI_IMAGE_LOADER`
    - `__DRI_IMAGE_LOOKUP`
    - `__DRI_USE_INVALIDATE`
    - `__DRI_BACKGROUND_CALLABLE`
  - EGL/WAYLAND provides `dri2_loader_extensions`
    - `__DRI_IMAGE_LOADER`
    - `__DRI_IMAGE_LOOKUP`
    - `__DRI_USE_INVALIDATE`
  - these functions are used
    - `__DRIimageLoaderExtensionRec` is for working with drawables
      - the loader is supposed to provide the backing storage, which is an
        array of `__DRIimage`, for a `__DRIdrawable`
    - `__DRIimageLookupExtensionRec` is for egl image import
      - e.g., when `glEGLImageTargetTexture2DOES` gets an egl image
    - `__DRIbackgroundCallableExtensionRec` is for glthread
      - egl/wayland does not support glthread?
- screen extensions
  - after a screen is created, `__DRIcoreExtensionRec::getExtensions` queries
    the screen extensions
  - GLX asks for `exts`
    - `__DRI2_RENDERER_QUERY`
    - `__DRI2_FLUSH`
    - `__DRI_IMAGE`
    - `__DRI2_INTEROP`
    - `__DRI2_CONFIG_QUERY`
  - GBM asks for `dri_core_extensions`
    - `__DRI2_FLUSH`
    - `__DRI_IMAGE`
  - EGL asks for `dri2_core_extensions` and `optional_core_extensions`
    - `__DRI2_FLUSH`
    - `__DRI_TEX_BUFFER`
    - `__DRI_IMAGE`
    - `__DRI2_CONFIG_QUERY`
    - `__DRI2_FENCE`
    - `__DRI2_BUFFER_DAMAGE`
    - `__DRI2_INTEROP`
    - `__DRI2_FLUSH_CONTROL`
    - `__DRI2_BLOB`
    - `__DRI_MUTABLE_RENDER_BUFFER_DRIVER`
    - `__DRI_KOPPER`
  - these functions are used
    - `__DRI2flushExtensionRec` is used when the loader is about to swap
      buffers or copy buffers, and needs an implicit flush
    - `__DRIimageExtensionRec` is for working with `__DRIimage`
      - it can allocate `__DRIimage`, which is `pipe_resource`
      - it can import texobj, rb, dmabuf, etc. as `__DRIimage`
      - it can map/umap `__DRIimage`, which is for gbm
      - it can blit between `__DRIimage`s, which is for prime blit
      - it can add a sync-file fd (e.g., from the display) to a `__DRIimage`,
        which is for android
    - `__DRI2interopExtensionRec` is for `EGL_MESA_gl_interop` and
      `GLX_MESA_gl_interop`
    - `__DRI2configQueryExtensionRec` queries driconf from the driver
    - `__DRI2fenceExtensionRec` is for various EGL fence extensions
    - `__DRI2bufferDamageExtensionRec` is for `EGL_KHR_partial_update`
    - `__DRI2flushControlExtensionRec` is for `EGL_KHR_context_flush_control`
    - `__DRI2blobExtensionRec` is for `EGL_ANDROID_blob_cache`
    - `__DRImutableRenderBufferDriverExtensionRec` is for
      `EGL_KHR_mutable_render_buffer`
- note that swrast uses different extensions

## `__DRI2_FLUSH`

- loaders need to flush bufferred rendering commands in a context in several
  cases
- GBM
  - `gbm_bo_unmap` might imply a blit that needs to be flushed
    - `gbm_dri_bo_unmap`
    - `__DRI2_FLUSH_CONTEXT`
- EGL/X11
  - `eglCopyBuffers` copies from a surface to a pixmap
    - `dri2_copy_buffers`
    - `dri2_egl_display_vtbl::copy_buffers`
    - `dri3_copy_buffers`
    - `loader_dri3_copy_drawable`
      - we need to flush such that the pending gl commands to the fake front
        are implicitly fenced when the xserver does the copy
  - `eglWaitClient` waits for the gl rendering
    - `dri2_wait_client`
    - flush with implied `__DRI2_FLUSH_DRAWABLE`
      - we need to flush such that the pending gl commands are implicitly
        fenced when the xserver does anything
  - `eglSwapBuffers`
    - `dri2_swap_buffers`
    - `dri2_egl_display_vtbl::swap_buffers`
    - `dri3_swap_buffers`
    - `dri3_swap_buffers_with_damage`
    - `loader_dri3_swap_buffers_msc`
    - `loader_dri3_vtable::flush_drawable`
    - `egl_dri3_flush_drawable`
    - `dri2_flush_drawable_for_swapbuffers`
    - `dri2_flush_drawable_for_swapbuffers_flags`
    - `__DRI2_FLUSH_DRAWABLE | __DRI2_FLUSH_INVALIDATE_ANCILLARY` and
      `__DRI2_THROTTLE_SWAPBUFFER`
- EGL/WAYLAND
  - `eglCopyBuffers` is not supported
  - `eglWaitClient` is the same as EGL/X11
  - `eglSwapBuffers`
    - `dri2_swap_buffers`
    - `dri2_egl_display_vtbl::swap_buffers`
    - `dri2_wl_swap_buffers`
    - `dri2_wl_swap_buffers_with_damage`
    - `dri2_flush_drawable_for_swapbuffers`, which is the same as in EGL/X11
- GLX
  - `glXCopySubBufferMESA` asks the xserver to copy from the back buffer to
    the front buffer
    - `__GLXDRIscreenRec::copySubBuffer`
    - `dri3_copy_sub_buffer`
      - `flush` is always true
    - `loader_dri3_copy_sub_buffer`
    - `__DRI2_FLUSH_DRAWABLE | __DRI2_FLUSH_CONTEXT` and
      `__DRI2_THROTTLE_COPYSUBBUFFER`
      - we need to flush such that the pending gl commands are implicitly
        fenced when the xserver does the copy
  - `glXWaitX` waits for the xserver rendering
    - `glx_context_vtable::wait_x`
    - `dri3_wait_x`
    - `loader_dri3_wait_x`
      - normally, the xserver commands are implicitly fenced and there is no
        wait needed
      - but a window always has a fake front in dri3 and we need to copy from
        the real front to the fake front
    - `loader_dri3_copy_drawable`
    - `__DRI2_FLUSH_DRAWABLE` and `__DRI2_THROTTLE_COPYSUBBUFFER`
      - we need to flush such that the pending gl commands to the fake front
        are implicitly fenced when the xserver does the copy
  - `glXWaitGL` waits for GL rendering
    - `glx_context_vtable::wait_gl`
    - `dri3_wait_gl`
    - `loader_dri3_wait_gl`
      - normally, the gl commands are implicitly fenced and there is no wait
        needed
      - but a window always has a fake front in dri3 and we need to copy from
        the fake front to the real front
    - `loader_dri3_copy_drawable`, same reason as other call sites
  - `glXSwapBuffers` or `glXSwapBuffersMscOML`
    - `__GLXDRIscreenRec::swapBuffers`
    - `dri3_swap_buffers`
      - `flush` may be true or false depending on whether the drawable is
        current
    - `loader_dri3_swap_buffers_msc`
    - `loader_dri3_vtable::flush_drawable`
    - `glx_dri3_flush_drawable`
    - `__DRI2_FLUSH_DRAWABLE` and `__DRI2_THROTTLE_SWAPBUFFER`
      - `__DRI2_FLUSH_CONTEXT` is set too when the drawable is current
  - `glFlush`, if frontbuffer rendering
    - `_mesa_Flush`
    - `_mesa_flush`
    - `st_glFlush`
    - `st_manager_flush_frontbuffer`
    - `pipe_frontend_drawable::flush_front`
    - `dri_st_framebuffer_flush_front`
    - `dri_drawable::flush_frontbuffer`
    - `dri2_flush_frontbuffer`
    - `__DRIimageLoaderExtensionRec::flushFrontBuffer`
    - `dri3_flush_front_buffer`
    - `__DRI2_FLUSH_DRAWABLE` and `__DRI2_THROTTLE_FLUSHFRONT`
- there is also `__DRI2flushExtensionRec::invalidate`
  - it tells the DRI driver to invalidate the backing storage of the drawable
  - EGL/X11 `eglQuerySurface`
    - `dri2_query_surface`
    - `dri2_egl_display_vtbl::query_surface`
    - `dri3_query_surface`
    - `loader_dri3_update_drawable_geometry`
    - invalidate if geometry change detected
  - EGL/X11 `eglSwapBuffers`
    - `dri2_swap_buffers`
    - `dri2_egl_display_vtbl::swap_buffers`
    - `dri3_swap_buffers`
    - `dri3_swap_buffers_with_damage`
    - `loader_dri3_swap_buffers_msc`
    - invalidate as the front/back is swapped
  - here is how gl validates the drawable
    - `st_manager_validate_framebuffers`
    - `pipe_frontend_drawable::validate`
    - `dri_st_framebuffer_validate`
    - `dri_drawable::allocate_textures`
    - `dri2_allocate_textures`
    - `dri_image_drawable_get_buffers`
    - `__DRIimageLoaderExtensionRec::getBuffers`
    - `loader_dri3_get_buffers` (egl/x11, glx) or `image_get_buffers` (egl/wl)
- `dri_flush_drawable` implements `__DRI2flushExtensionRec::flush`
  - it calls `dri_flush` with `__DRI2_FLUSH_DRAWABLE`
- `dri_flush` implements `__DRI2flushExtensionRec::flush_with_flags`
  - `_mesa_glthread_finish` waits for the gl thread to become idle
  - `__DRI2_FLUSH_CONTEXT` is mapped to `ST_FLUSH_FRONT`
  - `__DRI2_THROTTLE_SWAPBUFFER` is mapped to `ST_FLUSH_END_OF_FRAME`
  - `screen->throttle` is from `PIPE_CAP_THROTTLE` which is usually true
  - `st_context_flush` flushes the gl context
- `st_context_flush`
  - `ST_FLUSH_END_OF_FRAME` is mapped to `PIPE_FLUSH_END_OF_FRAME`
  - `ST_FLUSH_FENCE_FD` is mapped to `PIPE_FLUSH_FENCE_FD`
  - `ST_FLUSH_FRONT` causes `st_manager_flush_frontbuffer`

## Gallium DRI Megadriver

- `src/gallium/targets/dri/target.c` defines `__driDriverGetExtensions_<name>`
  for various drivers.  They return one of the following lists of extensions
  - `galliumdrm_driver_extensions` for hw drivers
  - `galliumsw_driver_extensions` for swrast
  - `dri_swrast_kms_driver_extensions` for `kms_swrast`
  - `galliumvk_driver_extensions` for zink
- to create the `pipe_screen`,
  - `driCreateNewScreen2`
  - `dri2_init_screen`
    - `pipe_loader_drm_probe_fd` is called to dup the fd, probe, and determine
      the `drm_driver_descriptor`
      - `loader_get_driver_for_fd` queries the driver name
      - `get_driver_descriptor` looks up `drm_driver_descriptor` from a
        statically-defined `driver_descriptors` array
        - the descriptors are defined via `DRM_DRIVER_DESCRIPTOR` 
      - it also sets `pipe_loader_ops` to `pipe_loader_drm_ops`
    - `pipe_loader_create_screen` creates the pipe screen
      - `pipe_loader_drm_create_screen`
      - `pipe_iris_create_screen`
      - `iris_drm_screen_create`
      - `iris_screen_create`
- kmsro
  - kmsro is used when the display uses one DRM driver and the GPU uses
    another DRM driver (e.g., mobile)
  - in `pipe_loader_drm_probe_fd`, we might fall back to
    `kmsro_driver_descriptor`
  - `kmsro_drm_screen_create` calls the gallium driver's screen create
    function with `struct renderonly`

## swrast

- GLX
  - driver loading
    - `__glXInitialize`
    - `driswCreateDisplay`
    - `driswCreateScreen`
    - `driswCreateScreenDriver` (`swrast` or `zink`)
    - `driOpenDriver`
    - `loader_open_driver`
  - asked driver extensions, `exts`
    - `__DRI_CORE`
    - `__DRI_SWRAST`
    - `__DRI_KOPPER`
    - `__DRI_COPY_SUB_BUFFER`
    - `__DRI_MESA`
  - provided loader extensions, `loader_extensions_shm`
    - `__DRI_SWRAST_LOADER`
    - `__DRI_KOPPER_LOADER`
  - asked screen extensions, `exts`
    - `__DRI_TEX_BUFFER`
    - `__DRI2_RENDERER_QUERY`
    - `__DRI2_FLUSH`
    - `__DRI2_CONFIG_QUERY`
- GBM
  - driver loading
    - `dri_device_create`
    - `dri_screen_create_sw` (try `zink`, `kms_swrast`, and `swrast` in order)
    - `dri_screen_create_for_driver`
    - `dri_open_driver`
    - `loader_open_driver`
  - asked driver extensions, `gbm_swrast_device_extensions`
    - same as hw
  - provided loader extensions, `gbm_dri_screen_extensions`
    - same as hw
  - asked screen extensions, none
- EGL
  - driver loading
    - `eglInitialize`
    - `dri2_initialize`
    - `dri2_initialize_x11`
    - `dri2_initialize_x11_swrast` (`zink` or `swrast`)
    - `dri2_load_driver_swrast`
    - `dri2_load_driver_common`
    - `dri2_open_driver`
    - `loader_open_driver`
  - asked driver extensions, `swrast_driver_extensions`
    - `__DRI_CORE`
    - `__DRI_MESA`
    - `__DRI_SWRAST`
    - `__DRI_CONFIG_OPTIONS`
  - provided loader extensions, `swrast_loader_extensions`
    - `__DRI_SWRAST_LOADER`
    - `__DRI_IMAGE_LOOKUP`
    - `__DRI_KOPPER_LOADER`
  - asked screen extensions, `swrast_core_extensions` and
    `optional_core_extensions`
    - `__DRI_TEX_BUFFER`
    - and optional ones
- swrast
  - the gallium dri megadriver provides `galliumsw_driver_extensions`, used by
    `swrast_dri.so`
- `kms_swrast`
  - the gallium dri megadriver provides  `dri_swrast_kms_driver_extensions`,
    used by `kms_swrast_dri.so`
  - this is preferred and is faster than `swrast_dri.so` by rendering to dumb
    bos directly (as opposed to shms)
  - this is only used with KMS-only kernel drivers, where mesa uses vgmem

## DRI gallium frontend

- DRI gallium frontend implements DRI interface over gallium interface
- `LIBGL_ALWAYS_SOFTWARE=1`
  - screen and context creation
    - `mesa->createNewScreen`
      - `driCreateNewScreen2`
      - `drisw_init_screen`
        - `pipe_loader_sw_probe_dri` calls `dri_create_sw_winsys` to create a
          `sw_winsys` with `drisw_lf`.  This builds the call path from pipe
          driver to winsys to dri frontend to the loader
      - `pipe_loader_create_screen`
      - `pipe_loader_create_screen_vk`
      - `pipe_loader_sw_create_screen`
      - `sw_screen_create_vk` uses the first of llvmpipe and softpipe
      - `sw_screen_create_named`
      - `llvmpipe_create_screen` creates a pipe screen from the winsys
    - `mesa->createContext`
      - `driCreateContextAttribs`
      - `dri_create_context`
      - `st_api_create_context`
      - `llvmpipe_create_context`
  - displaytarget creation
    - `st_framebuffer_validate`
    - `dri_st_framebuffer_validate`
    - `drisw_allocate_textures`
    - `llvmpipe_resource_create`
    - `llvmpipe_resource_create_front`
    - `llvmpipe_resource_create_all`
    - `llvmpipe_displaytarget_layout`
    - `dri_sw_displaytarget_create` allocates the backing storage either with
      `malloc` or with `alloc_shm`, depending on
      `drisw_loader_funcs::put_image_shm`
  - swapbuffers
    - `core->swapBuffers`
      - `driSwapBuffers`
      - `drisw_swap_buffers`
      - `drisw_copy_to_front`
      - `drisw_present_texture`
      - `llvmpipe_flush_frontbuffer`
      - `dri_sw_displaytarget_display`
      - `drisw_put_image`
      - `put_image`
      - `loader->putImage` sends the data to the compositor
- `MESA_LOADER_DRIVER_OVERRIDE=kms_swrast`
  - screen and context creation
    - `mesa->createNewScreen`
      - `driCreateNewScreen2`
      - `dri_swrast_kms_init_screen`
        - `pipe_loader_sw_probe_kms` calls `kms_dri_create_winsys` to create a
          `sw_winsys` with drm fd.
        - no `create_drawable` thus no drawable support
      - `pipe_loader_create_screen`
      - the rest is the same as swrast
    - `mesa->createContext` is the same as swrast
  - displaytarget creation
    - `gbm_bo_create`
      - `kms_swrast` has no drawable support, and this happens in gbm bo alloc
      - there is also no modifier support because llvmpipe lacks
        `resource_create_with_modifiers`
    - `gbm_dri_bo_create`
    - `loader_dri_create_image`
    - `dri2_create_image`
    - `dri2_create_image_common`
    - `llvmpipe_resource_create`
    - `llvmpipe_resource_create_front`
    - `llvmpipe_resource_create_all`
    - `llvmpipe_displaytarget_layout`
    - `kms_sw_displaytarget_create`
      - this uses `DRM_IOCTL_MODE_CREATE_DUMB` and requires a primary node
  - displaytarget map
    - `gbm_bo_map`
    - `gbm_dri_bo_map`
    - `dri2_map_image`
    - `pipe_texture_map`
    - `llvmpipe_transfer_map`
    - `llvmpipe_transfer_map_ms`
    - `llvmpipe_resource_map`
    - `kms_sw_displaytarget_map`
  - displaytarget unmap is similar
  - displaytarget export
    - `gbm_bo_get_fd_for_plane`
    - `gbm_dri_bo_get_plane_fd`
    - `dri2_query_image`
    - `dri2_query_image_by_resource_param`
    - `dri2_resource_get_param`
    - `llvmpipe_resource_get_param`
    - `llvmpipe_resource_get_handle`
    - `kms_sw_displaytarget_get_handle`
  - displaytarget import
    - `eglCreateImage` with `EGL_LINUX_DMA_BUF_EXT`
    - `_eglCreateImageCommon`
    - `dri2_create_image`
    - `dri2_create_image_khr`
    - `dri2_create_image_dma_buf`
    - `dri2_from_dma_bufs2`
    - `dri2_create_image_from_fd`
    - `dri2_create_image_from_winsys`
    - `llvmpipe_resource_from_handle`
    - `kms_sw_displaytarget_from_handle`
