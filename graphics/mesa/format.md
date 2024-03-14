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
