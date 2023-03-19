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
