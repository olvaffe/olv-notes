DRM AMD Display
===============

## Display

- display ip blocks
  - gfx6 uses `si_set_ip_blocks` and DCE 6.x
  - gfx7 uses `cik_set_ip_blocks` and DCE 8.x
  - gfx8 uses `vi_set_ip_blocks` and DCE 10.x/11.x
  - gfx9+ uses `amdgpu_discovery_set_display_ip_blocks`
  - newer chips use `dm_ip_block` rather than `dce_v*_ip_block`
- `DRM_IOCTL_MODE_ADDFB2` calls `drm_mode_addfb2_ioctl`
  - it calls `amdgpu_display_user_framebuffer_create` on amdgpu
  - `amdgpu_display_get_fb_info` extracts the tiling from the bo metadata
  - `convert_tiling_flags_to_modifier` converts tiling to modifiers
- `add_gfx9_modifiers`
  - `TILE_VERSION` is `AMD_FMT_MOD_TILE_VER_GFX9`
  - `TILE` is 
    - `AMD_FMT_MOD_TILE_GFX9_64K_D_X`
    - `AMD_FMT_MOD_TILE_GFX9_64K_D`
    - `AMD_FMT_MOD_TILE_GFX9_64K_S_X` (raven only)
    - `AMD_FMT_MOD_TILE_GFX9_64K_S` (raven only)
  - if `TILE` is `*_X`
    - `PIPE_XOR_BITS` depends on `num_pipes` and `num_se`
    - `BANK_XOR_BITS` additionally depends on `num_banks`
  - raven additionally supports `DCC`
    - `TILE` must be `AMD_FMT_MOD_TILE_GFX9_64K_S_X`
    - `DCC_INDEPENDENT_64B` is set
    - `DCC_MAX_COMPRESSED_BLOCK` is `AMD_FMT_MOD_DCC_BLOCK_64B`
    - if `DCC_RETILE`,
      - `RB` depends on `num_se` and `num_rb_per_se`
      - `PIPE` depends on `num_pipes`
  - raven2 additionally supports `DCC_CONSTANT_ENCODE`
- `add_gfx10_1_modifiers`
  - `TILE_VERSION` is `AMD_FMT_MOD_TILE_VER_GFX10`
    - `TILE` is
      - `AMD_FMT_MOD_TILE_GFX9_64K_R_X`
      - `AMD_FMT_MOD_TILE_GFX9_64K_S_X`
    - `PIPE_XOR_BITS` depends on `num_pipes`
    - if `DCC`
      - `TILE` must be `AMD_FMT_MOD_TILE_GFX9_64K_R_X`
      - `DCC_CONSTANT_ENCODE` is set
      - `DCC_INDEPENDENT_64B` is set
      - `DCC_MAX_COMPRESSED_BLOCK` is `AMD_FMT_MOD_DCC_BLOCK_64B`
      - `DCC_RETILE` is optional
  - `TILE_VERSION` is `AMD_FMT_MOD_TILE_VER_GFX9`
    - `TILE` is
      - `AMD_FMT_MOD_TILE_GFX9_64K_D`
      - `AMD_FMT_MOD_TILE_GFX9_64K_S`
