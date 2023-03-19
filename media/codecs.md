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
