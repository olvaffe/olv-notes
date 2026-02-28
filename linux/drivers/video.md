Kernel Video
============

## Configs

- `driver/video/Kconfig`
  - `drivers/auxdisplay/Kconfig`
    - it provides supports for char-based displays
    - userspace updates messages via `/dev/lcd` or sysfs
  - `drivers/char/agp/Kconfig`
    - it provides supports for agp bus, which is irrelevant
    - userspace configs gart via `/dev/agpgart`
  - `drivers/gpu/vga/Kconfig`
    - `CONFIG_VGA_SWITCHEROO` is for hybrid gpus
      - gpu drivers call `vga_switcheroo_register_client` to register as clients
      - mux driver calls `vga_switcheroo_register_handler` to register as the handler
      - even muxless, mux driver can still cut the power to dgpu
        - that is, after dgpu driver suspends the pci dev, mux driver can cut
          the power for greater saving
  - `drivers/gpu/host1x/Kconfig`
    - `CONFIG_TEGRA_HOST1X` provides a bus that gpu/media drivers register to
  - `drivers/gpu/ipu-v3/Kconfig`
    - `CONFIG_IMX_IPUV3_CORE` provides imx5/6 image processor unit support
  - `drivers/gpu/nova-core/Kconfig`
    - `CONFIG_NOVA_CORE` is a dep of `CONFIG_DRM_NOVA`
  - `drivers/gpu/drm/Kconfig`
  - `drivers/video/fbdev/Kconfig`
    - all drivers such as `CONFIG_FB_VESA` and `CONFIG_FB_EFI` are legacy
    - `drivers/video/fbdev/core/Kconfig`
      - `CONFIG_FB_CORE` is dep of `CONFIG_FRAMEBUFFER_CONSOLE`
        - display drivers call `register_framebuffer` to register to fb core,
          which calls `fbcon_fb_registered` automatically
      - `CONFIG_FB_DEVICE` provides `/dev/fb*` to userspace, which is legacy
  - `drivers/video/backlight/Kconfig`
    - `CONFIG_LCD_CLASS_DEVICE` provides `/sys/class/lcd` support
    - `CONFIG_BACKLIGHT_CLASS_DEVICE` provides `/sys/class/backlight` support
  - `drivers/video/console/Kconfig`
    - `CONFIG_DUMMY_CONSOLE` provides `dummy_con` to vt
      - it is the default backend
    - `CONFIG_FRAMEBUFFER_CONSOLE` provides `fb_con` to vt
    - `CONFIG_VGA_CONSOLE` provides `vga_con` to vt, which is irrelevant
  - `drivers/video/logo/Kconfig`
    - `CONFIG_LOGO` provides `fb_find_logo` to return a logo image
  - `drivers/gpu/trace/Kconfig`
    - `CONFIG_TRACE_GPU_MEM` provides `gpu_mem/gpu_mem_total` tracepoint
- `drivers/pci/Kconfig`
  - `CONFIG_VGA_ARB` keeps track of all `PCI_CLASS_DISPLAY_VGA` pci devices
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

## fbdev

- legacy configs
  - `CONFIG_FB` enables legacy fbdev drivers
  - `CONFIG_FB_DEVICE` exports fbdev to `/dev/fb*`
- modern configs
  - `CONFIG_FB_CORE`
  - `CONFIG_FRAMEBUFFER_CONSOLE_DETECT_PRIMARY`
- `fb_console_setup` parses `fbcon=`
- fbdev driver allocs and inits `fb_info`
  - `framebuffer_alloc` allocs an `fb_info`
  - `fb_alloc_cmap` allocs `info->cmap`
    - this is the color palette, usually 256 colors per channel
  - `info->skip_vt_switch` is typically true
- `register_framebuffer` registers an `fb_info`
  - `fb_device_create` creates a userspace device, if the legacy
    `CONFIG_FB_DEVICE` is enabled
  - `fb_info->pixmap` is initialized
  - `pm_vt_switch_required` enables/disables vt switch on suspend/resume
  - `fb_var_to_videomode` inits `fb_videomode` from `info->var`
  - `fb_add_videomode` adds the mode to `info->modelist`
  - `fbcon_fb_registered` registers the fbdev for vt console, if
    `CONFIG_FRAMEBUFFER_CONSOLE` is enabled
    - `fbcon_select_primary` checks if the fbdev is primary
      - it is determined by `video_is_primary_device` on x86
      - if primary, it updates `primary_device` and updates `con2fb_map_boot`
        to use this fbdev for all vts
      - otherwise, there is no primary device and `con2fb_map_boot` is all
        zeros, meaning the first fbdev is used for all vts
    - `do_fbcon_takeover` calls `do_take_over_console` from vt to enable the
      vt console
      - `fb_con` is the vt console ops

## fbdev

- In kernel, a fbdev is a VT driver.  That is how VT draws characters on screen.
- The first thing to do is to switch to a new VT and ask the VT not to draw to
  the screen
  - `ioctl(fd, VT_ACTIVATE, vtno)`
  - `ioctl(fd, KDSETMODE, KD_GRAPHICS)`
- Then we can open the fbdev device and draw at will
- Xorg
  - observed via `ps -ax -o sid,comm,tpgid,pid,tty` and the source
  - The server is in its own session.
  - It has the new VT as the controlling terminal
    - to set the new VT to graphics mode
    - to avoid other processes stealing the VT
  - It is also the foreground process of the new VT
  - Input events are received from evdev
    - the new VT is constantly flushed to keep its buffer clean
    - the new VT is made raw to avoid signals and others
  - It is also the foreground process of the old VT
    - it can receive Ctrl-C from the old VT
