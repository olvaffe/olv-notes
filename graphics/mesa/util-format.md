Mesa Formats
============

## Public Formats

- `VK_FORMAT`
  - when non-packed, channels are given from the lowest addr to the highest
    addr
    - endianness-neutral
    - e.g., `VK_FORMAT_B8G8R8_UNORM` has B in the lowest addr and R in the
      highest addr
  - when packed, channels are given from the MSB to the LSB
    - endianness-dependent
    - e.g., `VK_FORMAT_R5G6B5_UNORM_PACK16` has R in MSB and B in LSB
- `DRM_FORMAT` and `GBM_FORMAT`
  - channels are given from the highest addr to the lowest addr
  - endianness-neutral
  - e.g., `DRM_FORMAT_ARGB8888` has A in the highest addr and B in the lowest
    addr
    - it is officially defined as `[31:0] A:R:G:B 8:8:8:8 little endian`
    - that is, assuming little endian, A is in MSB and B is in LSB
    - iow, if we assume big endian, A will be in LSB and B will be in MSB
- `GL`
  - pixel transfers are defined by both `format` and `type`
    - for non-packed `type`, each value represents 1 component and is mapped
      to 1 channel in order
      - endianness-neutral
      - e.g., `GL_RGBA` and `GL_UNSIGNED_BYTE` has R in the lowest addr and A
        in the highest addr
      - that is, first byte is mapped to R, second byte is mapped to G, etc.
    - for packed `type`, each value represents multiple components and are
      mapped to channels in different rules
      - endianness-dependent
      - e.g., `GL_RGBA` and `GL_UNSIGNED_INT_8_8_8_8` has R in MSB and A in
        LSB
      - on the other hand, `GL_RGBA` and `GL_UNSIGNED_INT_8_8_8_8_REV` has R
        in LSB and A in MSB
  - `internalFormat` specifies which channels are stored and their size
    - the order is imlpementation-defined
    - e.g., `GL_RGBA8` merely says all 4 channels are stored and each has 8
      bits
- `HAL_PIXEL_FORMAT` and `AHARDWAREBUFFER_FORMAT`
  - when non-packed, the naming convention is the same as `VK_FORMAT`
  - when packed, there is no convention
    - `AHARDWAREBUFFER_FORMAT_R10G10B10A10_UNORM` is
      `VK_FORMAT_R10X6G10X6B10X6A10X6_UNORM_4PACK16`
    - `AHARDWAREBUFFER_FORMAT_R10G10B10A2_UNORM` is
      `VK_FORMAT_A2B10G10R10_UNORM_PACK32`
    - `AHARDWAREBUFFER_FORMAT_R5G6B5_UNORM` is `VK_FORMAT_R5G6B5_UNORM_PACK16`
- `WL_SHM_FORMAT`
  - this is the same as `DRM_FORMAT`
- `SkColorType`
  - there is no rule because some color types are misnamed
  - according to `docs/examples/Color_Type_*.cpp`
    - `kRGBA_1010102_SkColorType` is `VK_FORMAT_A2B10G10R10_UNORM_PACK32`
    - `kRGBA_8888_SkColorType` is `VK_FORMAT_A8B8G8R8_UNORM_PACK32`
    - `kBGRA_8888_SkColorType` is `VK_FORMAT_A8R8G8B8_UNORM_PACK32`
    - `kRGBA_F16_SkColorType` is `VK_FORMAT_R16G16B16A16_SFLOAT`
    - `kARGB_4444_SkColorType` is `VK_FORMAT_R4G4B4A4_UNORM_PACK16`
    - `kRGB_565_SkColorType` is `VK_FORMAT_R5G6B5_UNORM_PACK16`
  - according to `GrColorType` and `SkColorTypeToGrColorType`,
    - `kARGB_4444_SkColorType` should have been `kABGR_4444_SkColorType`
    - `kRGB_565_SkColorType` should have been `kBGR_565_SkColorType`
    - iow, `GrColorType` is fixed and has rules
      - when channel size is 8 or less, the format is packed
      - when packed, channels are given from the LSB to the MSB

## Internal Formats

- `PIPE_FORMAT` and `MESA_FORMAT`
  - when non-packed, channels are given from the lowest addr to the highest
    addr
    - endianness-neutral
    - e.g., `PIPE_FORMAT_B8G8R8A8_UNORM` is the same as
      `VK_FORMAT_B8G8R8A8_UNORM`
  - when packed, channels are given from the LSB to the MSB
    - endianness-dependent
    - e.g., `PIPE_FORMAT_B5G6R5_UNORM` is the same as
      `VK_FORMAT_R5G6B5_UNORM_PACK16`
- `__DRI_IMAGE_FORMAT`
  - the naming is the same as `DRM_FORMAT`

## Pipe Formats

- `u_format.yaml`
  - `name` is `enum pipe_format` without the `PIPE_FORMAT_` prefix
    - e.g., `B8G8R8A8_UNORM`
  - `layout` is `enum util_format_layout` without the `UTIL_FORMAT_LAYOUT_`
    prefix
    - `UTIL_FORMAT_LAYOUT_PLAIN` is when the block size is 1x1 and is plain
    - `UTIL_FORMAT_LAYOUT_SUBSAMPLED` is when the block is subsampled
      - still single plane
    - `UTIL_FORMAT_LAYOUT_S3TC` is S3TC/DXTC/BCn-compressed
    - `UTIL_FORMAT_LAYOUT_RGTC` is RGTC-compressed
    - `UTIL_FORMAT_LAYOUT_ETC` is ETC1/ETC2-compressed
    - `UTIL_FORMAT_LAYOUT_BPTC` is BPTC-compressed
    - `UTIL_FORMAT_LAYOUT_ASTC` is ASTC-compressed
    - `UTIL_FORMAT_LAYOUT_ATC` is ATC-compressed (AMD)
    - `UTIL_FORMAT_LAYOUT_PLANAR2` is for 2-plane
    - `UTIL_FORMAT_LAYOUT_PLANAR3` is for 3-plane
    - `UTIL_FORMAT_LAYOUT_FXT1` is FXT1-compressed
    - `UTIL_FORMAT_LAYOUT_OTHER` is for all other special cases
  - `sublayout` is for compression variants
    - ETC layout has ETC1 and ETC2 sublayouts
  - `colorspace` is `enum util_format_colorspace`
    - `UTIL_FORMAT_COLORSPACE_RGB`
    - `UTIL_FORMAT_COLORSPACE_SRGB`
    - `UTIL_FORMAT_COLORSPACE_YUV`
    - `UTIL_FORMAT_COLORSPACE_ZS`
  - `block` is the block size
    - `PLAIN` is always 1x1
    - `SUBSAMPLED` is always 2x1
      - all single-plane yuv formats are 4:2:2?
    - `S3TC` is always 4x4
    - `RGTC` is always 4x4
    - `ETC` is always 4x4
    - `BPTC` is always 4x4
    - `ASTC` has various block sizes
    - `ATC` is always 4x4
    - `PLANAR2` is mostly 1x1, but can be 4x1
    - `PLANAR3` is always 1x1
    - `FXT1` is always 8x4
    - `OTHER` is 1x1, 4x4, or 8x1
  - `channels` is channel types and sizes
    - they are given in memory order
      - because of how pipe formats are named, they match the order of the
        pipe format name
      - `Z32_FLOAT_S8X24_UINT` and `X32_S8X24_UINT` are special
    - the possible types are
      - `UN` and `SN` are unsigned/signed normalized
      - `UP` and `SP` are unsigned/signed pure integers
      - `X` is opaque
      - `F` is float-point
      - `H` is fixed-point
    - `PLAIN` describes channels in the same order as in the format name
    - `SUBSAMPLED` is somewhat random
    - `S3TC` is `X64` (no alpha) or `X128` (with alpha)
    - `RGTC` is `X64` (no alpha) or `X128` (with alpha)
    - `ETC` is `X64` (no alpha) or `X128` (with alpha)
    - `BPTC` is `X128`
    - `ASTC` is `X128`
    - `ATC` is `X64` (no alpha) or `X128` (with alpha)
    - `PLANAR2` is somewhat random
    - `PLANAR3` is somewhat random
    - `FXT1` is `X128`
    - `OTHER` is somewhat random
  - `swizzles` is channel swizzles
    - they are given in RGBA order
- `struct util_format_description`
  - most come from the yaml file, but some are derived
  - `block.bits` is the sum of channel sizes in bits
  - `nr_channels` is number of channels
  - `is_array` is true when all non-X channels have the same type
  - `is_bitmask` is true when all channels are non-fixed/non-float and
    `block.bits` is 8, 16, or 32
  - `is_mixed` is true when channels have different types
  - `is_unorm` is true if the name has `UNORM` or `SRGB` or is comporessed non-float
  - `is_snorm` is true if the name has `SNORM`
  - `channel` and `swizzle`
    - `UTIL_ARCH_BIG_ENDIAN` is totally nonsense
      - see the comment before how the parser derives `self.be_channels`
  - `srgb_equivalent` and `linear_equivalent`
- YUV formats
  - these are single-plane
    - `PIPE_FORMAT_UYVY`
    - `PIPE_FORMAT_VYUY`
    - `PIPE_FORMAT_YUYV`
    - `PIPE_FORMAT_YVYU`
    - `PIPE_FORMAT_R8G8_B8G8_UNORM`
    - `PIPE_FORMAT_G8R8_G8B8_UNORM`
    - `PIPE_FORMAT_G8R8_B8R8_UNORM`
    - `PIPE_FORMAT_R8G8_R8B8_UNORM`
    - `PIPE_FORMAT_B8R8_G8R8_UNORM`
    - `PIPE_FORMAT_R8B8_R8G8_UNORM`
    - `PIPE_FORMAT_G8B8_G8R8_UNORM`
      - `VK_FORMAT_G8B8G8R8_422_UNORM`
    - `PIPE_FORMAT_B8G8_R8G8_UNORM`
      - `VK_FORMAT_B8G8R8G8_422_UNORM`
    - `PIPE_FORMAT_Y210`
    - `PIPE_FORMAT_Y212`
    - `PIPE_FORMAT_Y216`
  - these are bi-planar
    - `PIPE_FORMAT_NV12`
    - `PIPE_FORMAT_NV21`
    - `PIPE_FORMAT_NV15`
    - `PIPE_FORMAT_NV20`
    - `PIPE_FORMAT_R8_G8B8_420_UNORM`
    - `PIPE_FORMAT_R8_B8G8_420_UNORM`
    - `PIPE_FORMAT_G8_B8R8_420_UNORM`
      - `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM`
    - `PIPE_FORMAT_R10_G10B10_420_UNORM`
    - `PIPE_FORMAT_R10_G10B10_422_UNORM`
    - `PIPE_FORMAT_X6G10_X6B10X6R10_420_UNORM`
      - `VK_FORMAT_G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`
    - `PIPE_FORMAT_X4G12_X4B12X4R12_420_UNORM`
      - `VK_FORMAT_G12X4_B12X4R12X4_2PLANE_420_UNORM_3PACK16`
    - `PIPE_FORMAT_NV16`
      - `VK_FORMAT_G8_B8R8_2PLANE_422_UNORM`
    - `PIPE_FORMAT_Y16_U16V16_422_UNORM`
      - `VK_FORMAT_G16_B16R16_2PLANE_422_UNORM`
    - `PIPE_FORMAT_P010`
    - `PIPE_FORMAT_P012`
    - `PIPE_FORMAT_P016`
      - `VK_FORMAT_G16_B16R16_2PLANE_420_UNORM`
    - `PIPE_FORMAT_P030`
  - these are tri-planar
    - `PIPE_FORMAT_YV12`
    - `PIPE_FORMAT_YV16`
    - `PIPE_FORMAT_IYUV`
    - `PIPE_FORMAT_R8_G8_B8_420_UNORM`
    - `PIPE_FORMAT_R8_B8_G8_420_UNORM`
    - `PIPE_FORMAT_G8_B8_R8_420_UNORM`
      - `VK_FORMAT_G8_B8_R8_3PLANE_420_UNORM`
    - `PIPE_FORMAT_R8_G8_B8_UNORM`
    - `PIPE_FORMAT_Y8_U8_V8_422_UNORM`
      - `VK_FORMAT_G8_B8_R8_3PLANE_422_UNORM`
    - `PIPE_FORMAT_Y8_U8_V8_444_UNORM`
      - `VK_FORMAT_G8_B8_R8_3PLANE_444_UNORM`
    - `PIPE_FORMAT_Y8_U8_V8_440_UNORM`
    - `PIPE_FORMAT_Y16_U16_V16_420_UNORM`
      - `VK_FORMAT_G16_B16_R16_3PLANE_420_UNORM`
    - `PIPE_FORMAT_Y16_U16_V16_422_UNORM`
      - `VK_FORMAT_G16_B16_R16_3PLANE_422_UNORM`
    - `PIPE_FORMAT_Y16_U16_V16_444_UNORM`
      - `VK_FORMAT_G16_B16_R16_3PLANE_444_UNORM`
  - hw support
    - hw typically lacks native support
      - its sampler either cannot sample yuv format or does not convert yuv to
        rgb
    - hw might support sampling yuv formats as rgb formats
      - its sampler treat yuv format as equivalent rgb format, and can sample
        without colorspace conversion
      - colorspace conversion requires driver emulation
    - hw might lack support completely
      - both sampling and colorspace conversion require driver emulation

## Swizzles

- `swizzle` is in RGBA order
  - `swizzle[0]` specifies the source of the R value
  - `swizzle[1]` specifies the source of the G value
  - `swizzle[2]` specifies the source of the B value
  - `swizzle[3]` specifies the source of the A value
- `util_format_compose_swizzles` composes two swizzles
  - hw typically supports only canonical formats and driver emulates other
    formats using swizzle (`swiz1`)
  - api typically allows users to swizzle (`swiz2`)
  - the final swizzle should be `swiz[i] = swiz1[swiz2[i]]`

## YCbCr

- `eglCreateImage` with `EGL_LINUX_DMA_BUF_EXT`
  - mesa egl parses the attrs and calls `dri2_from_dma_bufs`
    - `width` is from `EGL_WIDTH`
    - `height` is from `EGL_HEIGHT`
    - `fourcc` is from `EGL_LINUX_DRM_FOURCC_EXT`
    - `modifier` is from `EGL_DMA_BUF_PLANE0_MODIFIER_{LO,HI}_EXT`
    - `num_fds` is the memory plane count derived from the highest
      `EGL_DMA_BUF_PLANE{0,1,2,3}_MODIFIER_{LO,HI}_EXT`
    - `fds`, `strides`, `offsets` are from
      `EGL_DMA_BUF_PLANE{0,1,2,3}_{FD,OFFSET,PITCH}_EXT`
    - `yuv_color_space` is from `EGL_YUV_COLOR_SPACE_HINT_EXT`
    - `sample_range` is from `EGL_SAMPLE_RANGE_HINT_EXT`
    - `horizontal_siting` is from `EGL_YUV_CHROMA_HORIZONTAL_SITING_HINT_EXT`
    - `vertical_siting` is from `EGL_YUV_CHROMA_VERTICAL_SITING_HINT_EXT`
    - `flags` is from `EGL_PROTECTED_CONTENT_EXT`
  - `dri2_get_mapping_by_fourcc` maps the fourcc to `dri2_format_mapping`
    - `dri_fourcc` is the fourcc
    - if fourcc is tri-planar
      - `dri_format` is `__DRI_IMAGE_FORMAT_NONE`
      - `dri_components` is `__DRI_IMAGE_COMPONENTS_Y_U_V`
      - `pipe_format` is always `PIPE_FORMAT_IYUV`
        - it does not use the exact pipe formats, such as `PIPE_FORMAT_YV12`
      - `nplanes` is 3
      - `planes` describes the 3 planes in YUV order
        - `buffer_index` is similar to a swizzle
          - `planes[1].buffer_index = 2` means U is at memory plane 2
        - `width_shift` and `height_shift` are subsampling
        - `dri_format` is always `__DRI_IMAGE_FORMAT_R8`
    - if fourcc is bi-planar
      - `dri_format` is `__DRI_IMAGE_FORMAT_NONE`
      - `dri_components` is `__DRI_IMAGE_COMPONENTS_Y_UV`
      - `pipe_format` is the exact pipe format, such as `PIPE_FORMAT_NV12`
      - `nplanes` is 2
      - `planes` describes the 2 planes in Y and UV order
        - `buffer_index` is always 0 for Y and 1 for UV
        - `width_shift` and `height_shift` are subsampling
        - `dri_format` is the closest rgba `__DRI_IMAGE_FORMAT_*`
          - `__DRI_IMAGE_FORMAT_R8` for 8-bit Y
          - `__DRI_IMAGE_FORMAT_R16` for 16-bit Y
          - `__DRI_IMAGE_FORMAT_GR88` for 8-bit UV
          - `__DRI_IMAGE_FORMAT_GR1616` for 16-bit UV
          - `__DRI_IMAGE_FORMAT_NONE` if no corresponding dri format
    - if fourcc is packed and is not subsampled
      - `dri_format` is closest rgba `__DRI_IMAGE_FORMAT_*`
      - `dri_components` is
        - `__DRI_IMAGE_COMPONENTS_AYUV` if alpha
        - `__DRI_IMAGE_COMPONENTS_XYUV` if no alpha
      - `pipe_format` is the exact pipe format, such as `PIPE_FORMAT_AYUV`
      - `nplanes` is 1
      - `planes` describes the packed YUV plane
        - `buffer_index` is always 0
        - `width_shift` and `height_shift` are always 0
        - `dri_format` is the same as the iamge `dri_format`
    - if fourcc is packed and is subsampled
      - `dri_format` is `__DRI_IMAGE_FORMAT_NONE`
      - `dri_components` is
        - `__DRI_IMAGE_COMPONENTS_Y_XUXV`, if Y first
        - `__DRI_IMAGE_COMPONENTS_Y_UXVX`, if U/V first
      - `pipe_format` is the exact pipe format, such as `PIPE_FORMAT_YUYV`
      - `nplanes` is 2
      - `planes` describes the 2 planes in Y and UV order
        - `buffer_index` is always 0
        - `width_shift` and `height_shift` are subsampling
        - `dri_format` is the closest rgba `__DRI_IMAGE_FORMAT_*`
          - `__DRI_IMAGE_FORMAT_GR88` for 8-bit Y
            - it samples YU or YV, and discards chroma to keep just lumi
          - `__DRI_IMAGE_FORMAT_ARGB8888` or `__DRI_IMAGE_FORMAT_ABGR8888` for
            8-bit UV
            - it samples YUYV or UYVY, and discards lumo to keep just chroma
  - `dri2_get_modifier_num_planes` gets the expected memory plane count from
    the driver to validate against user-specified memory plane count
  - `dri_create_image_from_winsys` creates the image
    - if `map->pipe_format` is not natively supported, which is most likely,
      it may fallback to an alternative format
      - if fourcc is tri-planar,
        - `dri2_get_mapping_by_fourcc` somehow always maps them to
          `PIPE_FORMAT_IYUV`
        - but no driver supports `PIPE_FORMAT_IYUV` natively
        - `DRM_FORMAT_YUV420` may be remapped to
          `PIPE_FORMAT_R8_G8_B8_420_UNORM`
          - Y plane is considered R8
          - Cb plane is considered G8
          - Cr plane is considered B8
        - `DRM_FORMAT_YVU420` may be remapped to
          `PIPE_FORMAT_R8_B8_G8_420_UNORM`
          - Y plane is considered R8
          - Cr plane is considered B8
          - Cb plane is considered G8
      - if fourcc is bi-planar,
        - `DRM_FORMAT_NV12` may be remapped to `PIPE_FORMAT_R8_G8B8_420_UNORM`
          - Y plane is considered R8
          - CbCr plane is considered G8B8
        - `DRM_FORMAT_NV21` may be remapped to `PIPE_FORMAT_R8_B8G8_420_UNORM`
          - Y plane is considered R8
          - CrCb plane is considered B8G8
        - `DRM_FORMAT_NV16` may be remapped to `PIPE_FORMAT_R8_G8B8_422_UNORM`
        - `DRM_FORMAT_NV15` may be remapped to
          `PIPE_FORMAT_R10_G10B10_420_UNORM`
        - `DRM_FORMAT_NV20` may be remapped to
          `PIPE_FORMAT_R10_G10B10_422_UNORM`
      - if fourcc is packed and is subsampled
        - `DRM_FORMAT_YUYV` may be remapped to `PIPE_FORMAT_R8G8_R8B8_UNORM`
          - Y channel is considered R8
          - Cb channel is considered G8
          - Cr channel is considered B8
        - `DRM_FORMAT_YVYU` may be remapped to `PIPE_FORMAT_R8B8_R8G8_UNORM`
          - Y channel is considered R8
          - Cr channel is considered B8
          - Cb channel is considered G8
        - `DRM_FORMAT_UYVY` may be remapped to `PIPE_FORMAT_G8R8_B8R8_UNORM`
        - `DRM_FORMAT_VYUY` may be remapped to `PIPE_FORMAT_B8R8_G8R8_UNORM`
    - if the pipe format can be sampled, natively or after fallback,
      `resource_from_handle` is called from the last memory plane to the first
      memory plane
      - non-format planes are imported without a format
      - format planes are imported with the same pipe format but with
        width/height adjusted to subsampling
    - if the pipe format is not supported at all,
      - non-format planes are imported without a format
      - `map->nplanes` and `map->planes` are used in place of real format
        planes
      - it will be emulated and sampled plane-by-plane
- `glEGLImageTargetTexStorageEXT` with `GL_TEXTURE_EXTERNAL_OES`
  - `st_get_egl_image`
    - `dri_get_egl_image` looks up the image created by `dri2_from_dma_bufs`
    - `native_supported` is set depending on whether the pipe format is
      natively supported, or is a fallback or is emulated
  - `st_bind_egl_image` binds the image
    - the mesa format is based on the effective pipe format
- `glDraw*`
  - `st_prepare_draw` calls `st_validate_state` to validate the state
  - `st_update_fp` updates the fp state
  - `st_get_external_sampler_key` gets the fp variant key for external
    samplers
  - `st_get_fp_variant` returns the fp variant
    - `nir_lower_tex` is called with yuv lowering
      - this pass expects YUV to be in RGB channels respectively
      - this differs from vulkan where YUV are in GBR channels respectively
