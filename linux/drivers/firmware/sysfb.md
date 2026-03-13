Kernel sysfb
============

## x86

- `CONFIG_EFI_STUB`
  - `setup_graphics` inits `boot_params` for efifb
  - `parse_boot_params` copies `boot_params` to `sysfb_primary_display`
- `sysfb_init`
  - `screen_info_pci_dev` finds the pci device from `sysfb_primary_display`
  - `sysfb_parse_mode` converts screen info to `simplefb_platform_data`
  - if successful, `sysfb_create_simplefb` adds `simple-framebuffer` dev
    - the driver is drm `CONFIG_DRM_SIMPLEDRM` or fbdev `CONFIG_FB_SIMPLE`
  - else, it adds `efi-framebuffer` dev
    - the driver is drm `CONFIG_DRM_EFIDRM` or fbdev `CONFIG_FB_EFI`
- when the gpu driver calls `aperture_remove_conflicting_pci_devices` or
  `aperture_remove_conflicting_devices`,
  - `sysfb_disable` removes the platform device
