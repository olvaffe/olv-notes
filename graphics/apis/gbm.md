GBM
===

## API

- device functions
  - `gbm_create_device` creates a device from a drm fd (no ownership transfer)
  - `gbm_device_destroy`
  - `gbm_device_get_backend_name` returns `drm`
  - `gbm_device_get_fd` returns the drm fd
  - `gbm_device_is_format_supported` returns true if format/usage combo is supported
  - `gbm_device_get_format_modifier_plane_count` returns the plane count for a
    format/modifier combo
    - X11 uses this to skip intel compressed modifiers for scanout
    - `GBM_BO_USE_FRONT_RENDERING` solves a similar issue for freedreno
- bo creation functions
  - `gbm_bo_create` creates a bo with a format/flags combo
    - format is any fourcc, but the backend only supports a small set of
      non-planar formats
    - `GBM_BO_USE_SCANOUT` and `GBM_BO_USE_CURSOR` are for kms overlays
    - `GBM_BO_USE_RENDERING` is for gpu rendering
    - `GBM_BO_USE_WRITE` was for intel pwrite but is no longer used
    - `GBM_BO_USE_LINEAR` forces linear tiling
    - `GBM_BO_USE_PROTECTED` is for protected contents
    - `GBM_BO_USE_FRONT_RENDERING` disables compressions
    - note that there is
      - no media/camera support (only for display servers)
      - no texturing support (the bos cannot be sampled from?)
      - implied linear mapping support (was cheap with `I915_MMAP_OFFSET_GTT`
        which is not supported on newer intel hw)
  - `gbm_bo_create_with_modifiers` creates a bo with a format/modifier combo
    - no flags is a mistake
  - `gbm_bo_create_with_modifiers2` creates a bo with a format/modifier/flags combo
  - `gbm_bo_import` imports a bo with a format/modifier/flags combo
    - format is any fourcc, including planar ones
  - `gbm_bo_destroy`
- bo query functions
  - `gbm_bo_get_width`
  - `gbm_bo_get_height`
  - `gbm_bo_get_format`
  - `gbm_bo_get_modifier`
  - `gbm_bo_get_bpp` returns the bpp for a bo format
    - returns 0 if the format is planar
  - `gbm_bo_get_device`
  - `gbm_bo_set_user_data`
  - `gbm_bo_get_user_data`
  - plane functions
    - as noted in `gbm_bo_create`, this was designed for metadata planes, but
      can be repurposed for planar formats
    - `gbm_bo_get_plane_count` returns the plane count
      - up to 4 and includes metadata plane
    - `gbm_bo_get_fd_for_plane`
    - `gbm_bo_get_handle_for_plane`
    - `gbm_bo_get_stride_for_plane`
    - `gbm_bo_get_offset`
  - single-plane variants
    - `gbm_bo_get_fd`
    - `gbm_bo_get_handle`
    - `gbm_bo_get_stride`
- bo map functions
  - `gbm_bo_map` returns a linear view of a region of a bo
    - might require blit to readback / de-tile
      - as such, must set `GBM_BO_TRANSFER_READ` for the region to be
        initialized
    - all bos are assumed to be mappable
  - `gbm_bo_unmap`
  - `gbm_bo_write` was designed for cursor bo
- surface functions
  - for use with `EGL_PLATFORM_GBM_KHR`
  - `gbm_surface_create`
  - `gbm_surface_create_with_modifiers`
  - `gbm_surface_create_with_modifiers2`
  - `gbm_surface_destroy`
  - `gbm_surface_has_free_buffers`
  - `gbm_surface_lock_front_buffer`
  - `gbm_surface_release_buffer`
- global query functions
  - `gbm_format_get_name` returns the fourcc string of a fourcc format
- minigbm extensions
  - not always abi / api compatible
  - true disjoint bo support
    - `gbm_bo_map2` maps a regoin of a plane
      - useful when planes are in different gem bos
    - `gbm_bo_get_plane_size` is useless?
  - gralloc-like features
    - can allocate with planar formats
    - many more usage flags
      - media / camera
      - texturing
      - cpu mapping
      - ssbo
      - sensors

## GBM over DRI over Gallium

- To summarize,
  - with ycbcr, there are multiple planes pointing to the same bo
  - with modifier, there are multiple planes that might or might not point to
    the same bo
    - sometimes aux bo is required to be separated
  - ycbcr and modifier don't combine well
- `gbm_bo_create`
  - `__DRIimageExtension::createImage` to create a `__DRIimage`
  - `pipe_screen::resource_create` to create a `pipe_resource`
    - for ycbcr, it might or might not be chained depending on the driver
    - chained or not, all `pipe_resource` points to the same bo
    - that is format planes; modifiers might require memory planes
      - when they do, each `pipe_resource` in the chain might have a second or
      	even third bo for the metadata
      - drivers tend to limit modifiers to non-planar format to avoid
      	plane explosion
- `gbm_bo_create_with_modifiers`
  - `__DRIimageExtension::createImageWithModifiers` to create a `__DRIimage`
  - `pipe_screen::resource_create_with_modifiers` to create a `pipe_resource`
    - for ycbcr, it might or might not be chained depending on the driver
    - chained or not, all `pipe_resource` points to the same bo
- `gbm_bo_get_plane_count`
  - `__DRI_IMAGE_ATTRIB_NUM_PLANES`
  - `PIPE_RESOURCE_PARAM_NPLANES`
    - driver-dependent, but tend to report number of format planes OR number
      of memory planes
- `gbm_bo_get_stride_for_plane`
  - `__DRIimageExtension::fromPlanar` followed by `__DRI_IMAGE_ATTRIB_STRIDE`
  - `__DRIimageExtension::fromPlanar` creates a second `__DRIimage` pointing
    to the same `pipe_resource`
  - `PIPE_RESOURCE_PARAM_STRIDE`
- `gbm_bo_get_offset`
  - `__DRIimageExtension::fromPlanar` followed by `__DRI_IMAGE_ATTRIB_OFFSET`
  - `PIPE_RESOURCE_PARAM_OFFSET`
- `gbm_bo_get_handle_for_plane`
  - `__DRIimageExtension::fromPlanar` followed by `__DRI_IMAGE_ATTRIB_HANDLE`
  - `PIPE_RESOURCE_PARAM_HANDLE_TYPE_KMS`
    - with plane ignored by driver
- `gbm_bo_import` with `GBM_BO_IMPORT_FD_MODIFIER`
  - `__DRIimageExtension::createImageFromDmaBufs2` to create a `__DRIimage`
  - `pipe_screen::is_dmabuf_modifier_supported` to check if the
    format/modifier combo is supported
  - `pipe_screen::get_dmabuf_modifier_planes` to get the number of planes
    - for ycbcr with separated aux bo, iris returns 6; that is not goint to
      work
    - it is probably easier to assume that multi-planar and modifier do not
      work too well together
    - so one get plane count when multi-planar, or aux count when modifier
  - multiple `pipe_screen::resource_from_handle` to create `pipe_resource`
    for each plane from the last plane to the first plane
    - always a chained `pipe_resource` for ycbcr format
    - each plane has its own fd/stride/offset
      - different fds might refer to the same bo though

## minigbm

- to build,
  - `VERBOSE=1 CFLAGS=-DDRV_I915 MODE=dbg OUT=out DESTDIR=out/install make install`
  - for amdgpu, `CFLAGS="-DDRV_AMDGPU -DDRI_DRIVER_DIR=/usr/lib/dri"`
- `cros_gralloc_convert_format`
  - RGB
    - `HAL_PIXEL_FORMAT_RGBA_8888` to `DRM_FORMAT_ABGR8888`
    - `HAL_PIXEL_FORMAT_RGBX_8888` to `DRM_FORMAT_XBGR8888`
    - `HAL_PIXEL_FORMAT_RGB_888` to `DRM_FORMAT_BGR888`
    - `HAL_PIXEL_FORMAT_RGB_565` to `DRM_FORMAT_RGB565`
    - `HAL_PIXEL_FORMAT_BGRA_8888` to `DRM_FORMAT_ARGB8888`
    - `HAL_PIXEL_FORMAT_R8` to `DRM_FORMAT_R8`
    - `HAL_PIXEL_FORMAT_RGBA_FP16` to `DRM_FORMAT_ABGR16161616F`
    - `HAL_PIXEL_FORMAT_RGBA_1010102` to `DRM_FORMAT_ABGR2101010`
  - `HAL_PIXEL_FORMAT_RAW16` to `DRM_FORMAT_R16`
  - `HAL_PIXEL_FORMAT_BLOB` to `DRM_FORMAT_R8`
  - `HAL_PIXEL_FORMAT_Y8` to `DRM_FORMAT_R8`
  - `HAL_PIXEL_FORMAT_Y16` to `DRM_FORMAT_R16`
  - `HAL_PIXEL_FORMAT_YCBCR_P010` to `DRM_FORMAT_P010`
  - `HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED` to
    `DRM_FORMAT_FLEX_IMPLEMENTATION_DEFINED` to
    - `DRM_FORMAT_NV12` if camera
    - `DRM_FORMAT_XBGR8888` otherwise
  - `HAL_PIXEL_FORMAT_YCbCr_420_888` to `DRM_FORMAT_FLEX_YCbCr_420_888` to
    - `DRM_FORMAT_NV12`, the most common 420 format
  - `HAL_PIXEL_FORMAT_YV12` to `DRM_FORMAT_YVU420_ANDROID` to
    - `DRM_FORMAT_YVU420`
    - the difference is `DRM_FORMAT_YVU420_ANDROID` has android-specific plane
      layout requirements
- `cros_gralloc_convert_usage`
  - simple cases
    - `GRALLOC_USAGE_SW_READ_OFTEN` to `BO_USE_SW_READ_OFTEN`
    - `GRALLOC_USAGE_SW_READ_RARELY` to `BO_USE_SW_READ_RARELY`
    - `GRALLOC_USAGE_SW_WRITE_OFTEN` to `BO_USE_SW_WRITE_OFTEN`
    - `GRALLOC_USAGE_SW_WRITE_RARELY` to `BO_USE_SW_WRITE_RARELY`
    - `GRALLOC_USAGE_HW_TEXTURE` to `BO_USE_TEXTURE`
    - `GRALLOC_USAGE_HW_RENDER` to `BO_USE_RENDERING`
    - `GRALLOC_USAGE_HW_CAMERA_WRITE` to `BO_USE_CAMERA_WRITE`
    - `GRALLOC_USAGE_HW_CAMERA_READ` to `BO_USE_CAMERA_READ`
    - `GRALLOC_USAGE_RENDERSCRIPT` to `BO_USE_RENDERSCRIPT`
  - `GRALLOC_USAGE_HW_2D` to `BO_USE_RENDERING`
  - `GRALLOC_USAGE_HW_COMPOSER` to `BO_USE_SCANOUT | BO_USE_TEXTURE`
  - `GRALLOC_USAGE_HW_FB` to `BO_USE_NONE`
  - `GRALLOC_USAGE_EXTERNAL_DISP` to `BO_USE_NONE`
  - `GRALLOC_USAGE_PROTECTED` to `BO_USE_LINEAR`
  - `GRALLOC_USAGE_CURSOR` to `BO_USE_NONE`
  - `GRALLOC_USAGE_HW_VIDEO_ENCODER` to `BO_USE_HW_VIDEO_ENCODER | BO_USE_SW_READ_OFTEN`
  - `BUFFER_USAGE_VIDEO_DECODER` to `BO_USE_HW_VIDEO_DECODER`
  - `BUFFER_USAGE_GPU_DATA_BUFFER` to `BO_USE_GPU_DATA_BUFFER`
  - `BUFFER_USAGE_FRONT_RENDERING` to `BO_USE_FRONT_RENDERING`
- android common formats and flags 
  - surfaceflinger
    - all layers have `GraphicBuffer::USAGE_HW_TEXTURE` consumer usage
      - also `GraphicBuffer::USAGE_HW_COMPOSER`
      - potentially `GraphicBuffer::USAGE_PROTECTED` and
        `GraphicBuffer::USAGE_CURSOR`
    - composition engine asks for
      - requested format
      - `GRALLOC_USAGE_HW_RENDER`
      - `GRALLOC_USAGE_PROTECTED` if protected
    - framebuffers have
      - requested format
      - `GRALLOC_USAGE_HW_FB`
      - `GRALLOC_USAGE_HW_RENDER`
      - `GRALLOC_USAGE_HW_COMPOSER`
    - virtual displays have
      - requested format
      - `GRALLOC_USAGE_HW_COMPOSER`
    - refresh rate overlay asks for
      - `HAL_PIXEL_FORMAT_RGBA_8888`
      - `GRALLOC_USAGE_SW_WRITE_RARELY`
      - `GRALLOC_USAGE_HW_COMPOSER`
      - `GRALLOC_USAGE_HW_TEXTURE`
    - region sampling thread asks for
      - `PIXEL_FORMAT_RGBA_8888`
      - `GRALLOC_USAGE_SW_READ_OFTEN`
      - `GRALLOC_USAGE_HW_RENDER`
    - screen capture asks for
      - requested format
      - `GRALLOC_USAGE_SW_READ_OFTEN`
      - `GRALLOC_USAGE_SW_WRITE_OFTEN`
      - `GRALLOC_USAGE_HW_RENDER`
      - `GRALLOC_USAGE_HW_TEXTURE`
  - libgui
    - cpu consumer asks for `GRALLOC_USAGE_SW_READ_OFTEN`
    - gl consumer asks for `GraphicBuffer::USAGE_HW_TEXTURE`
    - surface asks for, when locking for cpu access,
      - `GRALLOC_USAGE_SW_READ_OFTEN`
      - `GRALLOC_USAGE_SW_WRITE_OFTEN`
