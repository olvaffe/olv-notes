Kernel DRM
==========

## Repos

- <https://drm.pages.freedesktop.org/maintainer-tools/>
- pull flow
  - <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/>
    - this is the upstream repo
  - <https://cgit.freedesktop.org/drm/drm>
    - Dave Airlie sends out the main pull request, `[git pull] drm for X.Y-rc1`,
      to Linus Torvalds to pull from
      `git://anongit.freedesktop.org/drm/drm tags/drm-next-YYYY-MM-DD`
    - Smaller fixes, `[git pull] drm fixes for X.Y-rcZ`, follow to
      pull from `git://anongit.freedesktop.org/drm/drm tags/drm-fixes-YYYY-MM-DD`
  - <https://cgit.freedesktop.org/drm/drm-intel>
    - Intel sends out the main pull request, `[PULL] drm-intel-next`,
      to Dave Airlie to pull from
      <git://anongit.freedesktop.org/drm/drm-intel tags/drm-intel-next-YYYY-MM-DD>
    - Smaller fixes, `[PULL] drm-intel-next-fixes`, follow to pull from
      `git://anongit.freedesktop.org/drm/drm-intel tags/drm-intel-next-fixes-YYYY-MM-DD`
  - <https://gitlab.freedesktop.org/agd5f/linux>
    - Alex Deucher sends out the main pull request, `[pull] amdgpu, amdkfd drm-next-X.Y`,
      to Dave Airlie to pull from
      `https://gitlab.freedesktop.org/agd5f/linux.git tags/amd-drm-next-X.Y-YYYY-MM-DD`
    - Smaller fixes, `[pull] amdgpu drm-fixes-X.Y`, follow to pull from
      `https://gitlab.freedesktop.org/agd5f/linux.git tags/amd-drm-fixes-X.Y-YYYY-MM-DD`
  - <https://gitlab.freedesktop.org/drm/msm>
    - Rob Clark sends out the main pull request, `[pull] drm/msm: drm-msm-next-YYYY-MM-DD for vX.Y`
      to Dave Airlie to pull from
      <https://gitlab.freedesktop.org/drm/msm.git tags/drm-msm-next-2023-04-10>
    - Smaller fixes, `[pull] drm/msm: drm-msm-fixes-YYYY-MM-DD for vX.Y-rcZ`, follow to pull from
      `https://gitlab.freedesktop.org/drm/msm.git tags/drm-msm-fixes-2023-05-17`
  - <https://cgit.freedesktop.org/drm/drm-misc>
    - Maarten Lankhorst sends out the main pull request, `[PULL] drm-misc-next`,
      to Dave Airlie to pull from
      `git://anongit.freedesktop.org/drm/drm-misc tags/drm-misc-next-YYYY-MM-DD`
    - Smaller fixes, `[PULL] drm-misc-next-fixes`, follow to pull from
      `git://anongit.freedesktop.org/drm/drm-misc tags/drm-misc-next-fixes-YYYY-MM-DD`
- drm
  - `drm/drm-next` branch is reset to `vX.Y-rc2` tag once the tag is created.
    The branch is then open for `Y+1` pull requests.
  - `drm/drm-fixes` branch is reset to `vX.Y-rcZ` tag once the tag is created.
    The branch is then open for `Y` fixes.
- drm-amd
  - `drm-amd/drm-next` branch is always open for `Y+1` changes
  - `drm-amd/amd-staging-drm-next` branch is always open for `Y` fixes
- drm-msm
  - `drm-msm/msm-next` branch is always open for `Y+1` changes.  It rebases
    against `drm/drm-next` occasionally.
  - `drm-msm/msm-fixes` branch is always open for `Y` fixes.  It rebases
    against `drm/drm-next` occasionally.

## Subdirectories

- desktop `DRIVER_MODESET+DRIVER_RENDER` drivers
  - `amd`, `i915`, `nouveau`, `radeon`, `xe`
- mobile `DRIVER_MODESET+DRIVER_RENDER` drivers
  - `exynos`, `loongson`, `msm`, `omapdrm`, `tegra`, `vc4`
- mobile `DRIVER_RENDER`-only drivers
  - `etnaviv`, `imagination`, `lima`, `panfrost`, `panthor`, `v3d`
- common `DRIVER_MODESET`-only drivers
  - `mediatek`, `rockchip`
- other `DRIVER_MODESET`-only drivers
  - `tiny`
  - `adp`, `arm`, `armada`, `aspeed`, `ast`, `atmel-hlcdc`, `fsl-dcu`,
    `gma500`, `gud`, `hisilicon`, `imx`, `ingenic`, `kmb`, `logicvc`, `mcde`,
    `meson`, `mgag200`, `mxsfb`, `pl111`, `renesas`, `solomon`, `sprd`, `sti`,
    `stm`, `sun4i`, `tidss`, `tilcdc`, `tve200`, `udl`, `xlnx`
- virtualized drivers
  - `hyperv`, `qxl`, `vgem`, `vboxvideo`, `virtio`, `vkms`, `vmwgfx`, `xen`
- display drivers
  - `bridge`, `panel`
- helpers
  - `clients`, `display`, `lib`, `scheduler`, `ttm`
- infra
  - `ci`, `tests`

## Drivers and Features

- summary
  - `DRIVER_GEM` is supported by all drivers
  - `DRIVER_MODESET` is supported by display-capable drivers
    - `DRIVER_ATOMIC` is supported by most
    - `DRIVER_CURSOR_HOTSPOT` is supported by virtualized
  - `DRIVER_RENDER` is supported by render-capable drivers
    - `DRIVER_SYNCOBJ` is supported by many
    - `DRIVER_SYNCOBJ_TIMELINE` is supported by few
    - `DRIVER_GEM_GPUVA` is supported only by a couple latest drivers
  - accelerators also use the drm subsystem
    - `DRIVER_COMPUTE_ACCEL` is supported by all
    - `DRIVER_GEM` is actually not supported by all
- caps
  - `DRM_CAP_DUMB_BUFFER` depends on `DRIVER_MODESET` and `driver->dumb_create`
    - `DRM_GEM_DMA_DRIVER_OPS` is used by ~41 drivers
    - `DRM_GEM_SHMEM_DRIVER_OPS` is used by ~12 drivers
    - `DRM_GEM_VRAM_DRIVER` is used by ~2 drivers
    - it is implemented directly by ~23 drivers
  - `DRM_CAP_VBLANK_HIGH_CRTC` depends on `DRIVER_MODESET`
  - `DRM_CAP_DUMB_PREFERRED_DEPTH` depends on `DRIVER_MODESET` and drivers
    - it is often 0, 16, or 24
  - `DRM_CAP_DUMB_PREFER_SHADOW` depends on `DRIVER_MODESET` and drivers
    - most drivers report false
  - `DRM_CAP_PRIME` is always `DRM_PRIME_CAP_IMPORT | DRM_PRIME_CAP_EXPORT`
  - `DRM_CAP_TIMESTAMP_MONOTONIC` is always true
  - `DRM_CAP_ASYNC_PAGE_FLIP` depends on `DRIVER_MODESET` and drivers
    - most drivers report false
    - this is flip outside of vsync (tearing)
  - `DRM_CAP_CURSOR_WIDTH` depends on `DRIVER_MODESET` and drivers
    - it is often 64
  - `DRM_CAP_CURSOR_HEIGHT` depends on `DRIVER_MODESET` and drivers
    - it is often 64
  - `DRM_CAP_ADDFB2_MODIFIERS` depends on `DRIVER_MODESET` and drivers
    - most drivers report true
  - `DRM_CAP_PAGE_FLIP_TARGET` depends on `DRIVER_MODESET` and drivers
    - most drivers report false
    - this is flip at Nth vsync
  - `DRM_CAP_CRTC_IN_VBLANK_EVENT` depends on `DRIVER_MODESET`
  - `DRM_CAP_SYNCOBJ` depends on `DRIVER_SYNCOBJ`
  - `DRM_CAP_SYNCOBJ_TIMELINE` depends on `DRIVER_SYNCOBJ_TIMELINE`
  - `DRM_CAP_ATOMIC_ASYNC_PAGE_FLIP` depends on `DRIVER_MODESET`,
    `DRIVER_ATOMIC`, and drivers
    - most drivers report false
- amd `amdgpu_kms_driver`
  - `DRIVER_ATOMIC | DRIVER_GEM | DRIVER_RENDER | DRIVER_MODESET | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE`
    - if no DC support or virtualized, `DRIVER_ATOMIC` is disabled
- amd `amdgpu_partition_driver`
  - `DRIVER_GEM | DRIVER_RENDER | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE`
- amd `amdgpu_xcp_driver`
  - `DRIVER_GEM | DRIVER_RENDER`
- arm `komeda_kms_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- arm `hdlcd_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- arm `malidp_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- armada `armada_drm_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- aspeed `aspeed_gfx_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- ast `ast_driver`
  - `DRIVER_ATOMIC | DRIVER_GEM | DRIVER_MODESET`
- atmel-hlcdc `atmel_hlcdc_dc_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- etnaviv `etnaviv_drm_driver`
  - `DRIVER_GEM | DRIVER_RENDER`
- exynos `exynos_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC | DRIVER_RENDER`
- fsl-dcu `fsl_dcu_drm_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- gma500 `driver`
  - `DRIVER_MODESET | DRIVER_GEM`
- gud `gud_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- hisilicon `hibmc_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- hisilicon `ade_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- hyperv `hyperv_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- i915 `i915_drm_driver`
  - `DRIVER_GEM | DRIVER_RENDER | DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE`
    - if system lacks display, `DRIVER_MODESET | DRIVER_ATOMIC` is disabled
- imagination `pvr_drm_driver`
  - `DRIVER_GEM | DRIVER_GEM_GPUVA | DRIVER_RENDER | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE`
- imx `dcss_kms_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- imx `imx_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- imx `imx_lcdc_drm_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- ingenic `ingenic_drm_driver_data`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- kmb `kmb_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- lima `lima_drm_driver`
  - `DRIVER_RENDER | DRIVER_GEM | DRIVER_SYNCOBJ`
- logicvc `logicvc_drm_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- loongson `lsdc_drm_driver`
  - `DRIVER_MODESET | DRIVER_RENDER | DRIVER_GEM | DRIVER_ATOMIC`
- mcde `mcde_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- mediatek `mtk_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- meson `meson_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- mgag200 `mgag200_driver`
  - `DRIVER_ATOMIC | DRIVER_GEM | DRIVER_MODESET`
- msm `msm_driver`
  - `DRIVER_GEM | DRIVER_RENDER | DRIVER_ATOMIC | DRIVER_MODESET | DRIVER_SYNCOBJ`
    - if system lacks display, `DRIVER_MODESET | DRIVER_ATOMIC` is disabled
- mxsfb `lcdif_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- mxsfb `mxsfb_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- nouveau `driver_stub`, `driver_pci`, and `driver_platform`
  - `DRIVER_GEM | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE | DRIVER_GEM_GPUVA | DRIVER_MODESET | DRIVER_RENDER`
    - `driver_pci` has experimental `DRIVER_ATOMIC`
- omapdrm `omap_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM  | DRIVER_ATOMIC | DRIVER_RENDER`
- panel `ili9341_dbi_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- panfrost `panfrost_drm_driver`
  - `DRIVER_RENDER | DRIVER_GEM | DRIVER_SYNCOBJ`
- panthor `panthor_drm_driver`
  - `DRIVER_RENDER | DRIVER_GEM | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE | DRIVER_GEM_GPUVA`
- pl111 `pl111_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- qxl `qxl_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_CURSOR_HOTSPOT`
- radeon `kms_driver`
  - `DRIVER_GEM | DRIVER_RENDER | DRIVER_MODESET`
- renesas `rcar_du_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- renesas `rzg2l_du_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- renesas `shmob_drm_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- rockchip `rockchip_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- solomon `ssd130x_drm_driver`
  - `DRIVER_ATOMIC | DRIVER_GEM | DRIVER_MODESET`
- sprd `sprd_drm_drv`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- sti `sti_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- stm `drv_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- sun4i `sun4i_drv_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tegra `tegra_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC | DRIVER_RENDER | DRIVER_SYNCOBJ`
    - if system lacks display, `DRIVER_MODESET | DRIVER_ATOMIC` is disabled
- tidss `tidss_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tilcdc `tilcdc_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `arcpgu_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- tiny `bochs_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `cirrus_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- tiny `gm12u320_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- tiny `hx8357d_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `ili9163_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `ili9225_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `ili9341_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `ili9486_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `mi0283qt_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `ofdrm_driver`
  - `DRIVER_ATOMIC | DRIVER_GEM | DRIVER_MODESET`
- tiny `panel_mipi_dbi_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `repaper_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `simpledrm_driver`
  - `DRIVER_ATOMIC | DRIVER_GEM | DRIVER_MODESET`
- tiny `st7586_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tiny `st7735r_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- tve200 `tve200_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`
- udl `driver`
  - `DRIVER_ATOMIC | DRIVER_GEM | DRIVER_MODESET`
- v3d `v3d_drm_driver`
  - `DRIVER_GEM | DRIVER_RENDER | DRIVER_SYNCOBJ`
- vboxvideo `driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC | DRIVER_CURSOR_HOTSPOT`
- vc4 `vc4_drm_driver`
  - `DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_GEM | DRIVER_RENDER | DRIVER_SYNCOBJ`
- vc4 `vc5_drm_driver`
  - `DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_GEM`
- vgem `vgem_driver`
  - `DRIVER_GEM | DRIVER_RENDER`
- virtio `driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_RENDER | DRIVER_ATOMIC | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE | DRIVER_CURSOR_HOTSPOT`
    - if host lacks display, `DRIVER_MODESET | DRIVER_ATOMIC` is disabled
- vkms `vkms_driver`
  - `DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_GEM`
- vmwgfx `driver`
  - `DRIVER_MODESET | DRIVER_RENDER | DRIVER_ATOMIC | DRIVER_GEM | DRIVER_CURSOR_HOTSPOT`
- xe `driver`
  - `DRIVER_GEM | DRIVER_RENDER | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE | DRIVER_GEM_GPUVA`
    - unless display is disabled, `DRIVER_MODESET | DRIVER_ATOMIC` too
- xen `xen_drm_driver`
  - `DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC`
- xlnx `zynqmp_dpsub_drm_driver`
  - `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC`

## Logging

- log macros
  - `drm_<level>` defines to `dev_<level>`
    - it takes a `drm_device` instead of a `device`
    - it adds a `[drm]` prefix to the format
  - `drm_dbg_<category>` calls `__drm_dev_dbg`
    - it takes a `drm_device` instead of a `device`
    - it inserts a `drm_debug_category`
  - `drm_dbg` defines to `drm_dbg_driver` as a shorthand
  - `drm_WARN*` defines to `WARN*`
    - it adds a `<drv> <dev>: [drm]` prefix to the format
- `drm_debug_category`
  - `DRM_UT_CORE` (0x1) is for core
  - `DRM_UT_DRIVER` (0x2) is for driver
  - `DRM_UT_KMS` (0x4) is for modesetting
  - `DRM_UT_PRIME` (0x8) is for prime (import)
  - `DRM_UT_ATOMIC` (0x10) is for atomic modesetting
  - `DRM_UT_VBL` (0x20) is for vblank (spammy)
  - `DRM_UT_STATE` (0x40) is for atomic state (before each commit)
  - `DRM_UT_LEASE` (0x80) is for leasing
  - `DRM_UT_DP` (0x100) is for DisplayPort
  - `DRM_UT_DRMRES` (0x200) is for drmm (managed resources)
- log functions
  - `drm_debug_enabled` returns if a debug category is enabled
  - `__drm_dev_dbg` calls `__drm_dev_vprintk`
    - it calls `dev_printk`
    - it adds a `[drm:<func>]` prefix
- printer functions
  - all printers can have custom prefix, etc.
  - `drm_*_printer` inits a printer
    - `drm_coredump_printer` copies to `drm_print_iterator` (a buffer)
    - `drm_seq_file_printer` prints with `seq_printf`
    - `drm_info_printer` prints with `dev_info`
      - it adds a `[drm]` prefix to the format
    - `drm_dbg_printer` prints with `__drm_dev_vprintk`
  - `drm_print_*` prints to a printer
- deprecated macros
  - `DRM_<level>` defines to `printk`
    - it adds a `[drm]` prefix to the format
  - `DRM_ERROR` defines to `__drm_err`
  - `DRM_DEBUG_<category>` calls `__drm_dev_dbg`
    - no `device`
    - it inserts a `drm_debug_category`
  - `DRM_DEV_<level>` defines to `drm_dev_printk`
  - `DRM_DEV_DEBUG_<category>` calls `__drm_dev_dbg`
    - it takes a `device`
    - it inserts a `drm_debug_category`
- deprecated functions
  - `__drm_err` calls `__drm_dev_vprintk` without a device
  - `drm_dev_printk` calls `__drm_dev_vprintk` with a device

## DRM core and /dev and /sys

- sysfs
  - for a PCI device, there is a `pci_dev at
    `/sys/devices/pci<domain>:<bus>/<domain>:<bus>:<device>.<function>
  - driver probe calls `drm_dev_init` to add one primary device and one render
    device under the root device.  They are in the `drm` class, which result in
    - `/sys/.../drm/card<minor>`
    - `/sys/.../drm/renderD<minor>`
  - also
    - `/sys/class/drm/card<minor>`
    - `/sys/class/drm/renderD<minor>`
- devfs
  - `/dev/dri/card<minor>`
  - `/dev/dri/renderD<minor>`
- internally, each of those userspace nodes is a `drm_minor`
  - on DRM core init, `register_chrdev` is called with major `DRM_MAJOR` (226)
  - driver probe calls `drm_dev_init`
    - it calls `drm_fs_inode_new` to create a new anon inode from the internal
      drm fs for the `drm_device`
      - the inode address space is shared by all opened files of the device
      - it is to simplify buffer eviction and mapping control
    - it calls `drm_minor_alloc` twice for primary and render `drm_minor`s,
      where primary node will be in [0, 63] and render node will be in [128,
      191].  `drm_sysfs_minor_alloc` allocates the devices the userspace will
      see
    - after driver probe initializes the hw, it calls `drm_dev_register` to
      register the minor devices to the kernel and set up `minor->drm_minor`
      mappings
- when `/dev/dri/card<minor>` is opened, because the major is `DRM_MAJOR`, it
  uses `drm_stub_fops`
  - from the inode, we have minor and can find the `drm_minor`
  - it calls `replace_fops` to replace the fops with the driver's
  - it then jump to driver's open, which is usually `drm_open`.  A `drm_file`
    is created as the private data of the filp

## Security

- any user in `video` group can open the primary device
- the user becomes master automatically if the primary device has no user
  - don't rely on this to run Xorg rootless
  - imagine VT swtiches, rootless Xorg cannot drop/acquire master
- any user can open the render node
- root-only ops
  - `DRM_IOCTL_SET_MASTER`
  - `DRM_IOCTL_DROP_MASTER`
- master-only ops
  - modesetting ioctls
- authenticated-only ops
  - `DRM_IOCTL_GEM_FLINK`
  - `DRM_IOCTL_GEM_OPEN`
  - not used in DRI3
- rendernode-allowed ops
  - `DRM_IOCTL_PRIME_HANDLE_TO_FD`
  - `DRM_IOCTL_PRIME_FD_TO_HANDLE`
  - `DRM_IOCTL_GEM_CLOSE`
  - and driver-specific execbuffer and alloc ops
  - rendernode-allowed ops must be explicitly whitelisted
  - while rendernode is assumed authenticated, the autenticated-only ops above
    are not on the white list
- `drm_ioctl_kernel` and ioctl flags
  - `DRM_ROOT_ONLY`
    - allow only if `capable(CAP_SYS_ADMIN)`
  - `DRM_AUTH`
    - allow only if `drm_is_render_client(file_priv)` or
      `file_priv->authenticated`
    - `authenticated` if root or master or authed by master
  - `DRM_MASTER`
    - allow only if `drm_is_current_master(file_priv)`
  - `DRM_RENDER_ALLOW`
    - when cleared, deny if `drm_is_render_client(file_priv)`
  - `DRM_UNLOCKED`
    - when cleared, lock `drm_global_mutex` if `DRIVER_LEGACY`
- ioctl hex val
  - according to `_IOC(dir,type,nr,size)` macro,
    - bit 0..7: `nr`
    - bit 8..15: `type`
      - always `DRM_IOCTL_BASE` (0x64) for drm ioctls
    - bit 16..29: `size`
      - `sizeof(arg)`
    - bit 30..31: `dir`
      - `_IOC_WRITE` is 1
      - `_IOC_READ` is 2
  - `DRM_IOCTL_GEM_CLOSE` is `0x40086409`
    - it expands to `DRM_IOW(0x09, struct drm_gem_close)`
    - `nr` is `0x09`
    - `type` is `0x64`
    - `size` is `sizeof(struct drm_gem_close)` which is 8
    - `nr` is `0x01` which is `_IOC_WRITE`

## `drm_exec`

- before job submission, drivers need to lock all bos to update their states
  atomically
- if there is a concurrent submit happening on another queue, and the
  concurrent submit needs to lock some common bos in a different order, the
  locking must be done carefully to avoid deadlock
- `drm_exec_init` prepares `drm_exec`
- `drm_exec_until_all_locked(&exec){...;drm_exec_retry_on_contention(&exec)}`
  is the retry loop
  - `drm_exec_until_all_locked` essentially expands to a jump label followed
    by `while (drm_exec_cleanup(&exec))`
  - `drm_exec_retry_on_contention` expands to a jump to the label if
    contention happens
  - `drm_exec_cleanup` returns false only when all objects have been locked
    - otherwise, it unlocks all objects and returns true to restart
- inside the retry loop is usually another loop to lock objects one-by-one
  - `drm_exec_prepare_obj` calls `drm_exec_lock_obj` and
    `dma_resv_reserve_fences`
- `drm_exec_fini` unlocks all objs and cleans up `drm_exec`
