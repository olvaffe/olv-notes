GBM
===

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
