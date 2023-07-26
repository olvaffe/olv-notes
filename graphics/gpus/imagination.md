Imagination PowerVR
===================

## History

- Series1 in 1996-1997
  - licensed to NEC
  - DirectX 3.0
- Series2 in 1998-1999
  - licensed to NEC
  - DirectX 6.0
  - used in Dreamcast
- Series3 in 2000-2002
  - licensed to STMicro
  - DirectX 6.0
- Series4 in 2001
  - very successful on mobile
- Series5 in 2005-2010
  - SGX
  - OpenGL ES 2.0
  - OpenGL 2.1 to 3.2
  - Direct3D 9.0c to 10.1
  - Series5XT in 2009-2010
    - multi-core variant
    - used in PlayStation Vita
  - Series5XE in 2014
    - low-end
- Series6 in 2012-2013
  - Rogue
  - Vulkan 1.1
  - OpenGL ES 3.1
  - OpenGL 3.2
  - OpenCL 1.2
  - Direct3D 10.0
  - Series6XE in 2014
    - entry-level
    - Direct3D 9.0 L3
  - Series6XT in 2014
    - optimized power consumption
    - OpenGL 3.3
    - GX6xx0
- Series7 in 2014
  - still Rogue
  - Vulkan 1.1
  - OpenGL ES 3.1 and Android Extension Pack
  - OpenCL 1.2 embedded profile
  - Direct3D 9.0 L3
  - Series7XE in 2014
    - entry-level
  - Series7XT in 2014
    - multi-core
  - Series7XT Plus in 2016
    - Vulkan 1.1
    - OpenGL ES 3.2
    - OpenGL 4.4
    - OpenCL 2.0
    - Direct3D 12
- Series8 in 2016-2018
  - Rogue
  - Series8XT
  - Series8XE
  - Series8XEP
- Series9 in 2017-2018
  - Rogue
  - Series9XT
  - Series9XM
  - Series9XE
  - Series9XTP
  - Series9XMP
  - Series9XEP
- IMG A-Series in 2019
- IMG B-Series in 2020
- IMG C-Series in 2021

## Architectures

- PowerVR Rogue
- PowerVR Furian
- IMG A-Series
- PowerVR Photon

## Implementations

- MediaTek
  - 2015, MT8173, Series6XT GX6250
  - 2016, MT8176, Series6XT GX6250
  - 2017, Helio X30 (MT6799), Series7XT GT7400 Plus
  - 2017, MT6739, Series8XE GE8100
  - 2017, MT8167, Series8XE GE8300
  - 2018, Helio A20 (MT6761D), Series8XE GE8300
  - 2018, Helio P22 (MT6762), Series8XE GE8320
  - 2018, Helio A22 (MT6762M), Series8XE GE8320
  - 2018, Helio P35 (MT6765), Series8XE GE8320
  - 2018, Helio P90, Series9XM GM9446
  - 2019, MT6731, Series8XE GE8100
  - 2020, Helio A25, Series8XE GE8320
  - 2020, Helio G25, Series8XE GE8320
  - 2020, Helio G35, Series8XE GE8320
  - 2020, Helio P95, Series9XM GM9446
  - 2022, Dimensity 930 (MT6855 MT6855V/AZA), IMG BXM-8-256
  - 2023, Dimensity 7020, IMG BXM-8-256

## Chromebook hana

- ebuilds
  - `virtual/vulkan-icd` depends on `media-libs/vulkan-loader` and
    `media-libs/img-ddk-bin`
  - `virtual/opengles` depends on `virtual/img-ddk` and `media-libs/mesa-img`
  - `virtual/img-ddk` depends on `media-libs/img-ddk-bin`
  - these are public ebuilds and `media-libs/img-ddk-bin` provides prebuilt
    binaries
    - on the device, `media-libs/img-ddk` is installed which means internal
      ebuilds build from source code
- public source code
  - kernel driver is at
    <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/refs/heads/chromeos-5.15/drivers/gpu/drm/img-rogue/>
  - mesa dri driver is at
    <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/media-libs/mesa-img/files/0001-Add-pvr-dri-driver.patch>
    - it builds `pvr_dri.so` which loads proprietary `libpvr_dri_support.so`
    - there are `DRM_FORMAT_MOD_PVR_FBCDC_8x8_V7` and
      `DRM_FORMAT_MOD_PVR_FBCDC_16x4_V7`
    - `FBCDC` stands for frame buffer compression and decompression
- vulkaninfo
  - `/etc/vulkan/icd.d/icdconf.json` points to `libVK_IMG.so`
    - `libusc.so` is Universal Shading Cluster? compiler backend?
    - `libufwriter.so` is UniFlex writer? compiler IR?
    - `libsrv_um.so` is user-mode service? OS abstraction?
  - PowerVR Rogue GX6250
  - Vulkan 1.3
  - `VK_EXT_image_drm_format_modifier`
  - `VK_IMG_filter_cubic`
  - `VK_IMG_format_pvrtc`
    - Deprecated: Both PVRTC1 and PVRTC2 are slower than standard image
      formats on PowerVR GPUs, and support will be removed from future
      hardware
- `wflinfo -p null -a gles2`
  - strace says it loads these libraries
    - `pvr_dri.so`
    - `libpvr_dri_support.so`
    - `libGLESv2_PVR_MESA.so`
    - `libsrv_um.so`
    - `libglslcompiler.so`
    - `libusc.so`
  - PowerVR Rogue GX6250
  - OpenGL ES 3.2
