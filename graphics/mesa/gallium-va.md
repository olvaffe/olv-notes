Mesa and VAAPI
==============

## Quick Basics

- `vlVaCreateConfig` supports 3 entrypoints
  - `VAEntrypointVLD` is mapped to `PIPE_VIDEO_ENTRYPOINT_BITSTREAM`
  - `VAEntrypointEncSlice` is mapped to `PIPE_VIDEO_ENTRYPOINT_ENCODE`
  - `VAEntrypointVideoProc` is mapped to `PIPE_VIDEO_ENTRYPOINT_PROCESSING`
- when decoding,
  - `vlVaRenderPicture` with different buffer types
    - first `VASliceDataBufferType` calls `pipe_video_codec::begin_frame`
    - every `VASliceDataBufferType` calls `pipe_video_codec::decode_bitstream`
    - every `VAProcPipelineParameterBufferType` calls
      `pipe_video_codec::process_frame` for postprocessing
  - `vlVaEndPicture`
    - always calls `pipe_video_codec::end_frame`
    - if `PIPE_VIDEO_CAP_REQUIRES_FLUSH_ON_END_FRAME`, always calls
      `pipe_video_codec::flush`
  - `vlVaSyncSurface`
    - calls `pipe_video_codec::get_decoder_fence`
- radeonsi vcn decoding
  - `radeon_dec_begin_frame` maps the gpu buffer for writing
    - `radeon_decoder` maintains a dynamic array of gpu buffers
  - `radeon_dec_decode_bitstream` copies the bistream into the gpu buffer
    - `radeon_decoder` grows the dynamic array as needed
  - `radeon_dec_end_frame` submits the job to the gpu
    - `send_cmd_dec` builds cs
    - `flush` submits cs with `PIPE_FLUSH_ASYNC`
      - `amdgpu_cs_flush` queues the submit to the `cs` thread which submits
        the job to the kernel in `amdgpu_cs_submit_ib`
      - `PIPE_FLUSH_ASYNC` means `amdgpu_cs_flush` does not wait for the `cs`
        thread
    - `picture->fence` is associated with the submit
      - this updates `surf->fence` set up in `vlVaEndPicture`
    - `dec->prev_fence` is set to `picture->fence`
  - `radeon_dec_flush` is nop and is not called by va anyway
  - `radeon_dec_get_decoder_fence` waits the fence
    - `amdgpu_fence_wait`

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

## Flushes

- some operations imply gpu jobs that need to be flushed
  - `vlVaUnmapBuffer` may xfer into the buffer and need a flush
  - `vlVaPutImage` may xfer into the surface and need a flush
  - `vlVaPutSurface` copies with gpu and needs a flush
  - `vlVaHandleSurfaceAllocate` clears the buffer and needs a flush
  - `vlVaRenderPicture`, if doing postprocessing using gpu, needs a flush
- `vlVaEndPicture` calls `pipe_video_codec::flush`
