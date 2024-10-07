Kernel Video
============

## Configs

- drm, `CONFIG_DRM`
  - `CONFIG_DRM_FBDEV_EMULATION` enables fbdev on top of drm
    - drm drivers call `drm_fbdev_{shmem,ttm,dma}_setup` to enable fbdev
      emulation
      - they are nop when this config is enabled
    - this implies `CONFIG_FRAMEBUFFER_CONSOLE`
- backlight, `CONFIG_BACKLIGHT_CLASS_DEVICE`
  - it enables backlight class
- vt console
  - `CONFIG_FRAMEBUFFER_CONSOLE` register a fbdev-based `consw` to vt,
    allowing vt to render using the fbdev
- legacy agp, `CONFIG_AGP`
  - it enbbles the legacy `/dev/agpgart`
- legacy hybrid graphics, `CONFIG_VGA_SWITCHEROO`
  - the drm driver calls `vga_switcheroo_register_client`
  - on older devices, the dgpu has a mux for the display
    - when dgpu is enabled, dgpu is powered on and outputs to display
    - when igpu is used, dgpu is powered off and igpu outputs to display
  - on newer devices, there is no mux and this does powering on/off
  - userspace writes to `/sys/kernel/debug/vgaswitcheroo/switch` to do the
    switch
    - it relies on debugfs?  probably should never be used..
    - <https://wiki.archlinux.org/title/PRIME> instead
- legacy fbdev drivers
  - `CONFIG_FB` is for legacy fbdev drivers
  - it implies `CONFIG_FB_DEVICE` which enables the legacy `/dev/fb*`
