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

- <https://drmdb.emersion.fr/>
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
