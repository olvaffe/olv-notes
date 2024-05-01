Khronos Data Format
===================

## Specs

- <https://registry.khronos.org/DataFormat/specs/1.3/dataformat.1.3.html>

## ETC2

- <https://registry.khronos.org/DataFormat/specs/1.3/dataformat.1.3.html#ETC2>
  - RGB ETC2 is for linear/srgb RGB data
  - RGBA ETC2 is for linear/srgb RGBA data, where the RGB part is exactly the
    same as RGB ETC2 and the A part is similar to R11 EAC
  - R11 EAC is for one-channel signed/unsigned data
  - RG11 EAC is for two-channel signed/unsigned data, where each channel is
    exactly the same as R11 EAC
  - RGB ETC2 with "punchthrough" alpha is similar to RGB ETC2 but with a bit
    to indicate opaque/transparent
- for every 4x4 texel block,
  - RGB ETC2 takes 64 bits
  - RGBA ETC2 takes 128 bits
  - R11 EAC takes 64 bits
  - RG11 EAC takes 128 bits
- RGB ETC2
  - there are 5 modes
    - if bit 33 is 0, it's individual mode
    - if bit 33 is 1,
      - it's T mode if R is out-of-range
      - it's H mode if G is out-of-range
      - it's planar mode if B is out-of-range
      - otherwise, it's differential mode
  - individual mode
    - if bit 32 is 0, the 4x4 block is splitted into two 2x4 subblocks
    - if bit 32 is 1, the 4x4 block is splitted into two 4x2 subblocks
    - bit 40..63 are the base colors of the 2 subblocks, where each channel of
      each sublock takes 4 bits
    - bits 34..39 are the modifier tables, where each subblock takes 3 bits to
      select one of the 8 modifier tables
    - bit 0..31 are pixel indices, where each of the 16 texels takes 2 bits to
      select one of the 4 modifiers in the modifier table
    - IOW,
      - there is a 4-bit base color for each channel of each of the 2
        subblocks
      - there is a 3-bit table selector for each of the 2 sublocks
      - there is a 2-bit modifier selector for each of the 16 texels
  - differential mode
    - it is similar to the individual mode
    - rather than individual base colors for the 2 subblocks, the base color
      of the first subblock takes 5 bits for each channel.  The base color of
      the second subblock takes 3 bits for each channel and are summed to the
      base color of the first subblock
  - T and H modes
    - in both modes, 4 paint colors are derived from the higher 32 bits
    - each of the 16 texels takes 2 bits to select one of the paint colors
  - planar mode
    - there is a RGB676 base color (19 bits)
    - there are two RGB676 secondary colors (38 bits)
    - texel `(x, y)` of the 16 texels has value
      `base + (secondary_1 - base) * x / 4 + (secondary_2 - base) * y / 4`
- RGBA ETC2
  - the first 64 bits are for A
    - bit 56..63 are the 8-bit base color
    - bit 52..55 are the 4-bit multiplier
    - bit 48..51 are the 4-bit modifier table index, which selects one of the
      16 modifier tables
    - bit 0..47 are the 3-bit modifier indices for each of the 16 texels,
      which selects one of the 8 modifiers in the modifier table
    - `alpha = base + multiplier * modifier`
  - the second 64 bits are for RGB
    - they are the same as RGB ETC2
- R11 EAC
  - it is similar to the A part of RGBA ETC2
- RG11 EAC
  - the first and the second 64 bits are exactly the same as R11 EAC
- RGB ETC2 with "punchthrough" alpha
  - it is similar to RGB ETC2, except there is no individual mode
  - when bit 33 is 0, it can mean transparent in certain conditions
- <https://github.com/Themaister/Granite/blob/master/vulkan/texture/texture_decoder.cpp>
  - EAC
    - only unsigned support
    - `dispatch_kernel_eac`
      - push contants: width, height
      - specialization: `VK_FORMAT_EAC_R11G11_UNORM_BLOCK ? 2 : 1`
      - dispatch: `(width+7)/8, (height+7)/8, 1`
        - each workgroup handles four 4x4 texel blocks
    - <https://github.com/Themaister/Granite/blob/master/assets/shaders/decode/eac.comp>
      - `coord` is in texels and is the coord in the uncompressed tex
      - `tile_coord` is in texel blocks and is the coord in the compressed tex
      - `pixel_coord` is in texels and is the coord within a texel block
      - `linear_pixel` is linearized `pixel_coord`
      - `decode_eac_alpha` decodes the 64-bit value
        - see R11 EAC above
  - ETC2
    - `dispatch_kernel_etc2`
      - push contants: width, height
      - specialization: 0 (RGB ETC2), 1 (RGB ETC2 with punchthrough alpha), or 8 (RGBA ETC2)
      - dispatch: `(width+7)/8, (height+7)/8, 1`
        - each workgroup handles four 4x4 texel blocks
    - <https://github.com/Themaister/Granite/blob/master/assets/shaders/decode/etc2.comp>
      - `build_coord` is the same as in EAC
