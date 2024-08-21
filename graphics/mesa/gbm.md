GBM
===

## Origin

- GBM was created for EGL on DRM
  - main users were wayland compositors and xserver
  - compositors wanted to
    - allocate gbm bos and assign them to hw planes
    - use EGL/GLES to render to gbm bos
    - for client shms, upload them as texutres
    - for client dma-bufs, import them as texutres or assign them to hw planes
  - xserver additionally wanted to
    - use EGL/GLES to accelerate 2D rendering
- EGL integration
  - `EGL_PLATFORM_GBM_KHR`
    - a native window is a `gbm_surface`
      - this is for apps, not for compositors
    - a nativep pixmap is `gbm_bo`
  - to import a client dma-buf to EGL,
    - `gbm_bo_import` and then `eglCreateImage(EGL_NATIVE_PIXMAP_KHR)`
    - this was before `EGL_EXT_image_dma_buf_import`
- designed for compositors
  - `gbm_bo_create` can only allocate RGBA bos in DRI implementation
  - `gbm_bo_import` supports YUV formats on the other hand
  - has use flags for
    - scanout and cursor, to assign bos to hw planes
    - gpu rendering, which is almost always set
      - unless the compositor is sw-based
    - linear, which was added for prime
  - no use flags for
    - cpu mapping, which is assumed
    - gpu texturing, which is assumed by the gpu rendering flag?
    - other hw ips such as codec, camera, sensor, etc.

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
- `gbm_device_is_format_supported`
  - `__DRIimageExtension::queryDmaBufModifiers`
  - `dri2_query_dma_buf_modifiers` calls `pipe_screen::is_format_supported`
    and `pipe_screen::query_dmabuf_modifiers`
- `gbm_device_get_format_modifier_plane_count`
  - `__DRIimageExtension::queryDmaBufFormatModifierAttribs` with
    `__DRI_IMAGE_FORMAT_MODIFIER_ATTRIB_PLANE_COUNT`
  - `dri2_query_dma_buf_format_modifier_attribs` calls
    `pipe_screen::is_dmabuf_modifier_supported` and
    `pipe_screen::get_dmabuf_modifier_planes`
    - `DRM_FORMAT_MOD_LINEAR` and `DRM_FORMAT_MOD_INVALID` are considered
      well-known (no metadata plance and is equal to format plane count) and
      do not call into the pipe driver
- `gbm_bo_create`
  - `gbm_format_to_dri_format` maps fourcc to dri format
    - it only maps rgba formats and has no ycbcr support
  - `__DRIimageExtension::createImage` to create a `__DRIimage`
    - `dri2_create_image` uses `dri2_get_mapping_by_format` to map the dri
      format which does not support ycbcr either
  - `pipe_screen::resource_create` to create a `pipe_resource`
    - while `resource_create` can be used to allocate ycbcr resources, gbm
      does not suppor them
    - for ycbcr, it might or might not be chained depending on the driver
    - that is format planes; modifiers might require memory planes
      - when they do, each `pipe_resource` in the chain might have a second or
      	even third bo for the metadata
        - which drivers?
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

## Clients

- the api has 3 main categories
  - `gbm_device_*`
    - create/destory/query
    - `gbm_device_is_format_supported`
    - `gbm_device_get_format_modifier_plane_count`
  - `gbm_surface_`
    - for use with `EGL_KHR_platform_gbm`
  - `gbm_bo_*`
    - create/destory/queries
    - `gbm_bo_import`
    - `gbm_bo_get_handle` / `gbm_bo_get_handle_for_plane`
    - `gbm_bo_get_fd` / `gbm_bo_get_fd_for_plane`
    - `gbm_bo_write`
    - `gbm_bo_map`
- x11
  - xserver
    - `gbm_device_get_format_modifier_plane_count`
    - no surface
    - bo create/import/export
    - no bo map/write
  - xf86-video-amdgpu delta is empty
- wayland
  - weston
    - no device query
    - use surface
    - bo create/import/export/write
    - no bo map
  - wlroots
    - no device query
    - no surface
    - bo create/export
    - no bo import/map/write
- apps over kms
  - kmscube
    - no device query
    - use surface
    - bo create/export/map
    - no bo import/write
  - mpv
    - no device query
    - use surface
    - bo create/export/map
    - no bo import/write
  - sdl
    - `gbm_device_is_format_supported`
    - use surface
    - bo create/export/map/write
    - no bo import

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
- GEM handle refcount
  - each bo has a gem handle, `bo->handle.u32`
  - when a bo is allocated, exported as a dma-buf, and imported as another bo,
    - we will have two bos sharing the same gem handle
    - we must not close the gem handle until both bos are destroyed
  - `drv->buffer_table` maps gem handles to refcounts
  - `drv_bo_destroy` calls `bo_destroy` only when the refcount reaches 0
    - it calls `bo_release` right away, to free `bo->priv` which is not shared
      with other bos
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
- `Allocator` implements `BnAllocator`
  - `Allocator::init` calls `cros_gralloc_driver::get_instance`
    - it scans drm render nodes, calls `drv_create`, and returns the first
      `driver`
  - `Allocator::isSupported` calls `cros_gralloc_driver::is_supported`
    - `drv_resolve_format_and_use_flags` returns the format and use flags
    - `drv_get_combination` checks if the combo is supported
    - `drv_get_max_texture_2d_size` checks against the max img size
  - `Allocator::allocate2` calls `cros_gralloc_driver::allocate`
    - `drv_bo_create` allocates the `bo`
    - `cros_gralloc_handle` wraps the exported `bo`
      - `drv_bo_get_num_planes`
      - `drv_bo_get_plane_fd`
      - `drv_bo_get_plane_stride`
      - `drv_bo_get_plane_offset`
      - `drv_bo_get_plane_size`
      - `drv_bo_get_width`
      - `drv_bo_get_height`
      - `drv_bo_get_format`
      - `drv_bo_get_tiling`
      - `drv_bo_get_format_modifier`
      - `drv_bo_get_use_flags`
      - `drv_bytes_per_pixel_from_format`
      - `drv_bo_get_total_size`
      - note that `enable_metadata_fd` is always true
        - dynamically-sized metadata (e.g., name) and mutable metadata (e.g.,
          dataspace) are stored in shmem rather than embedded into the handle
        - it prefers dma-heap due to selinux policy
    - `cros_gralloc_buffer::create` wraps `cros_gralloc_handle`
      - this is mainly to initialize the metadata
- `CrosGrallocMapperV5` implements `IMapperV5Impl`
  - `CrosGrallocMapperV5::importBuffer` calls `cros_gralloc_driver::retain`
    - `drv_bo_import` imports the handle
    - `buffers_` and `handles_` track the buffer/handle
  - `CrosGrallocMapperV5::lock` calls `cros_gralloc_driver::lock`
    - it waits on the in-fence
    - it looks up the buffer and calls `cros_gralloc_buffer::lock`
    - `drv_bo_map`
      - recursive mapping calls `drv_bo_invalidate` instead
    - `drv_bo_get_plane_offset`
  - `CrosGrallocMapperV5::unlock` calls `cros_gralloc_driver::unlock`
    - `drv_bo_flush_or_unmap`
  - `CrosGrallocMapperV5::flushLockedBuffer` calls
    `cros_gralloc_driver::flush`
    - `drv_bo_flush`
  - `CrosGrallocMapperV5::rereadLockedBuffer`
    calls`cros_gralloc_driver::invalidate`
    - `drv_bo_invalidate`
  - `CrosGrallocMapperV5::getMetadata` only supports the std metadata
  - `CrosGrallocMapperV5::setMetadata` only supports the std metadata
  - `CrosGrallocMapperV5::dumpBuffer` dumps one buffer
  - `CrosGrallocMapperV5::dumpAllBuffers` dumps all buffers
  - `CrosGrallocMapperV5::getReservedRegion` returns the client reserved
    region
