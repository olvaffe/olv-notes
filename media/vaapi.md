Codecs
======

## Hardware Acceleration

- VA-API
  - <https://github.com/intel/libva>
  - open api designed by intel
  - encode and decode
  - drivers are installed to `/usr/lib/dri/<driver>_drv_video.so`
    - gallium amd
    - gallium nvidia
    - gallium virgl
    - gallium d3d12
    - non-mesa intel
      - new: <https://github.com/intel/media-driver>
      - old: <https://github.com/intel/intel-vaapi-driver>
    - vdpau adapter
    - nvdec adapter
  - `LIBVA_DRIVERS_PATH` to specify an alternative driver path
- VDPAU
  - <https://gitlab.freedesktop.org/vdpau/libvdpau>
  - open api designed by nvidia
  - decode only
  - drivers are installed to `/usr/lib/vdpau/libvdpau_<driver>.so`
    - gallium amd
    - gallium nvidia
    - gallium virgl
    - gallium d3d12
    - vaapi adapter
    - proprietary amd
    - proprietary nvidia
- Vulkan Video
  - open api designed by khronos
  - encode and decode
  - supported drivers
    - mesa amd and intel
    - proprietary amd
    - proprietary nvidia
- AMF
  - <https://gpuopen.com/advanced-media-framework/>
  - open api designed by amd
  - encode and decode
  - supported drivers
    - proprietary amd
- NVDEC/NVENC
  - <https://developer.nvidia.com/nvidia-video-codec-sdk>
  - proprietary api
  - encode and decode
  - supported drivers
    - proprietary nvidia

## VA-API: API and Gallium


- display init
  - `vaGetDisplayDRM` returns a `VADisplay` from a drm fd
    - this creates a `VADisplayContext` and a `VADriverContext` internally
    - `VADisplay` is `VADisplayContext`
  - `vaSetDriverName` overrides the driver library to load
  - `vaSetInfoCallback` sets info message callback
  - `vaSetErrorCallback` sets error message callback
  - `vaInitialize` initializes `VADisplay`
    - this loads the driver and calls `__vaDriverInit_<major>_<minor>`
    - gallium `VA_DRIVER_INIT_FUNC`
      - `vl_drm_screen_create` creates the pipe screen
      - `pipe_create_multimedia_context` creates the pipe context
      - `vtable` is statically-initialized
  - `vaTerminate` terminates `VADisplay`
    - gallium `vlVaTerminate` destroys the pipe context and the pipe screen
- display queries
  - `vaDisplayIsValid` returns true if `VADisplay` is valid
  - `vaQueryVendorString` returns the vendor string
  - `vaGetLibFunc` calls `dlsym` on the driver library (for private extensions)
- display attrs
  - `vaMaxNumDisplayAttributes` and `vaQueryDisplayAttributes` enumerate and
    get the current values of supported `VADisplayAttribute`s
  - `vaGetDisplayAttributes` gets the current values of gettable attrs
  - `vaSetDisplayAttributes` sets the current values of settable attrs
    - for comparison, other objects such as configs and surfaces are created
      with an array of attrs
      - their attrs cannot be modified after creation
- display profiles and profile entrypoints
  - `vaMaxNumProfiles` and `vaQueryConfigProfiles` enumerate supported
    `VAProfile`s
  - `vaMaxNumEntrypoints` and `vaQueryConfigEntrypoints` enumerate supported
    `VAEntrypoint`s of a `VAProfile`
- configs (profile/entrypoint pairs)
  - `VAConfigAttribTypeMax` and `vaGetConfigAttributes` enumerate supported
    `VAConfigAttrib`s for a profile/entrypoint pair
    - they are for use with config creation
    - `VAConfigAttribRTFormat` is a bitmask of `VA_RT_FORMAT_*`
      - these only indicate yuv/rgb, bpp, and subsampling
      - they are not enough if external/cpu access is needed
  - `vaCreateConfig` creates a `VAConfigID` from a `VAConfigAttrib` array
  - `vaDestroyConfig` destroys a config
  - `vaMaxNumConfigAttributes` and `vaQueryConfigAttributes` query info from
    a `VKConfigID`
    - they return `VAProfile`, `VAEntrypoint`, and `VAConfigAttrib`s
  - `vaQueryProcessingRate` queries the processing rate
- surfaces
  - `vaQuerySurfaceAttributes` enumerates supported `VASurfaceAttrib`s of a
    `VAConfigID`
    - they are for use with surface creation
    - gallium returns multiple `VASurfaceAttribPixelFormat`, one for each
      supported drm format compatible with the config RT format
    - gallium returns one `VASurfaceAttribDRMFormatModifiers`, to indicate a
      `VADRMFormatModifierList` may be specified
      - there is no way to query supported modifiers however
  - `vaCreateSurfaces` creates surfaces from a `VASurfaceAttrib` array
    - the mem type is `VA_SURFACE_ATTRIB_MEM_TYPE_VA` by default, and gallium
      calls `pipe_context::create_video_buffer` to allocate
    - if mem type is set to `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2`, gallium
      calls `pipe_context::resource_from_handle` to import
  - `vaDestroySurfaces` destroys surfaces
  - `vaSyncSurface` and `vaSyncSurface2` block for pending operations
    - when decoding, gallium calls `pipe_video_codec::get_decoder_fence`,
      which waits for the associated decode fence
  - `vaQuerySurfaceStatus` queries surface status (business)
  - `vaQuerySurfaceError` gets detailed error when `vaSyncSurface` fails
  - `vaExportSurfaceHandle` exports a surface for external access
- contexts
  - `vaCreateContext` creates a `VAContextID` from both `VAConfigID` and
    `VASurfaceID`
  - `vaDestroyContext` destroys a context
- buffers
  - `vaCreateBuffer` and `vaCreateBuffer2` creates a `VABufferID`
    - it has a `VABufferType` to specify its purpose
    - gallium mallocs a cpu memory
  - `vaDestroyBuffer`
  - `vaMapBuffer` maps a buffer for cpu access
  - `vaUnmapBuffer`
  - `vaBufferSetNumElements`
  - `vaAcquireBufferHandle` locks a buffer for external access
  - `vaReleaseBufferHandle`
  - `vaSyncBuffer` blocks for pending operations
- processing
  - `vaBeginPicture` prepares rendering
  - `vaRenderPicture` renders
    - when gallium sees `VAPictureParameterBufferType`, it calls
      `pipe_context::create_video_codec` to create the codec on demand
    - when gallium sees `VASliceDataBufferType`, it calls
      `pipe_video_codec::decode_bitstream` to copy the data from the
      malloc-based `VABufferID`s to gpu memory
  - `vaEndPicture` ends rendering
    - gallium calls `pipe_video_codec::end_frame` to start processing
  - `vaCopy` copies between buffers or surfaces
- image
  - `vaMaxNumImageFormats` and `vaQueryImageFormats` query supported
    `VAImageFormat`s for `VAImage`
  - `vaCreateImage` creates a `VAImage` for cpu access
  - `vaDestroyImage`
  - `vaSetImagePalette`
  - `vaGetImage` copies from a surface to an image
  - `vaPutImage` copies from an image to a surface
  - `vaDeriveImage` creates a image from a surface (zero-copy) for cpu access
- VPP (video post-processing)
  - `vaQueryVideoProcFilters` queries filters from a `VAContextID`
  - `vaQueryVideoProcFilterCaps` queries filter caps from a `VAContextID` and
    a `VAProcFilterType`
  - `vaQueryVideoProcPipelineCaps` queries the pipeline caps
- protected contents
  - `vaCreateProtectedSession`
  - `vaDestroyProtectedSession`
  - `vaAttachProtectedSession`
  - `vaDetachProtectedSession`
  - `vaProtectedSessionExecute`
- MF context
  - `vaCreateMFContext` creates a `VAMFContextID`
  - `vaMFAddContext` adds a `VAContextID` to a `VAMFContextID`
  - `vaMFReleaseContext` removes a `VAContextID` from a `VAMFContextID`
  - `vaMFSubmit`
- strings
  - `vaBufferTypeStr` returns strings for `VABufferType`
  - `vaConfigAttribTypeStr` returns strings for `VAConfigAttribType`
  - `vaEntrypointStr` returns strings for `VAEntrypoint`
  - `vaErrorStr` returns user-friendly strings for `VAStatus`
  - `vaProfileStr` returns strings for `VAProfile`
  - `vaStatusStr` returns strings for `VAStatus`
- legacy subpicture
  - `vaMaxNumSubpictureFormats` returns the max number of `VAImageFormat`s for
    `VASubpictureID`
  - `vaQuerySubpictureFormats` queries available `VAImageFormat`s for
    `VASubpictureID`
  - `vaCreateSubpicture` creates a `VASubpictureID` from a `VAImageID`
  - `vaDestroySubpicture`
  - `vaSetSubpictureImage` binds `VASubpictureID` to a different `VAImageID`
  - `vaSetSubpictureChromakey`
  - `vaSetSubpictureGlobalAlpha`
  - `vaAssociateSubpicture` assocaites `VASubpictureID` with `VASurfaceID`
    - when `vaPutSurface` is called to present to X11 , the surface is
      composited with the subpicture
  - `vaDeassociateSubpicture`

## VA-API: Intel

- public symbols are exported by `MEDIAAPI_EXPORT`
  - `VA_DRV_INIT_FUC_NAME` expands to the driver entrypoint
  - the other public symbols extensions specific to intel
- driver initialization
  - `DdiMedia__Initialize` is called from the driver entrypoint
  - `DdiMedia_InitMediaContext` sets `apoDdiEnabled`
    - this seems to be a pretty new feature
  - `DdiMedia_LoadFuncion` or `MediaLibvaInterface::LoadFunction` initializes
    the vtable depending on `apoDdiEnabled`
- `mos_bufmgr_gem_init` inits a `mos_bufmgr` over DRM fd
  - `bo_alloc` points to `mos_gem_bo_alloc`
  - `bo_alloc_for_render` points to `mos_gem_bo_alloc_for_render`
  - `bo_alloc_userptr` points to `mos_gem_bo_alloc_userptr`
  - `bo_alloc_tiled` points to `mos_gem_bo_alloc_tiled`
  - public methods
    - `mos_bo_alloc`
    - `mos_bo_alloc_for_render`
    - `mos_bo_alloc_userptr`
    - `mos_bo_alloc_tiled`
    - `mos_gem_bo_alloc_internal`
    - `mos_bo_gem_create_from_prime`
    - `mos_bo_gem_export_to_prime`
- surface creation
  - `vaCreateSurfaces`
  - `DdiMedia_CreateSurfaces2`
  - `DdiMedia_CreateRenderTarget`
  - `DdiMediaUtil_CreateSurface`
  - `DdiMediaUtil_AllocateSurface`
    - `mos_bo_gem_create_from_prime`
    - `mos_bo_alloc_userptr`
    - `mos_bo_alloc`
    - `mos_bo_alloc_tiled`
- surface export
  - `vaExportSurfaceHandle`
  - `DdiMedia_ExportSurfaceHandle`
  - `mos_bo_gem_export_to_prime`
- image creation
  - `vaCreateImage`
  - `DdiMedia_CreateImage`
  - `DdiMediaUtil_CreateBuffer`
  - `DdiMediaUtil_AllocateBuffer`
  - `mos_bo_alloc`
- buffer creation
  - `vaCreateBuffer`
  - `DdiMedia_CreateBuffer`
  - `DdiDecode_CreateBuffer`, `DdiEncode_CreateBuffer`, or
    `DdiVp_CreateBuffer`
