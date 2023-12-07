DRM vgem
========

## vgem

- `vgem_driver`
  - `driver_featues` is `DRIVER_GEM | DRIVER_RENDER`
    - `DRIVER_GEM` is set for all modern drivers
    - `DRIVER_RENDER` creates `/dev/dri/renderDX` in addition to
      `/dev/dri/cardY`
  - `ioctls` is `vgem_ioctls`, with 2 ioctls
    - `vgem_fence_attach_ioctl`
    - `vgem_fence_signal_ioctl`
  - `fops` is `DEFINE_DRM_GEM_FOPS`
    - all fops (open, close, ioctl, poll, read, mmap) are standard
  - prime and dumb are `DRM_GEM_SHMEM_DRIVER_OPS`
  - `gem_create_object` is `vgem_gem_create_object`
    - it allocates `drm_gem_shmem_object` and sets `obj->map_wc`
- fencing
  - `vgem_fence_attach_ioctl` adds a ro or rw dma-fence to a bo and returns an
    id for the fence
    - `dma_fence_init` is called with `vgem_fence_ops`
    - all fences are on different timelines (`dma_fence_context_alloc`) and
      have seqno 1
    - a timer is setup to make sure the dma-fence signals after 10s
  - `vgem_fence_signal_ioctl` signals a fence immediately
    - it calls `dma_fence_signal`
- `DRM_CAP_DUMB_BUFFER`
  - when a driver supports `dumb_create`, `DRM_CAP_DUMB_BUFFER` is true
    - `DRM_GEM_SHMEM_DRIVER_OPS` uses `drm_gem_shmem_dumb_create`
  - it enables
    - `DRM_IOCTL_MODE_CREATE_DUMB`
    - `DRM_IOCTL_MODE_MAP_DUMB`
    - `DRM_IOCTL_MODE_DESTROY_DUMB`
- prime
  - `DRM_GEM_SHMEM_DRIVER_OPS` supports both export and import
  - export uses the standard `drm_gem_prime_handle_to_fd`
    - mmap of an exported dma-buf uses the standard `drm_gem_prime_mmap`
  - import uses the standard `drm_gem_prime_fd_to_handle`
    - `drm_gem_shmem_prime_import_sg_table` imports from the dma-buf's
      `sg_table`

## vgem usecases

- hw dma-buf testing
  - we can test dma-buf export/import between vgem and a hw driver
- zero-copy software rendering with kms-only driver
  - a kms-only driver has `DRIVER_MODESET` but no `DRIVER_RENDER`
  - mesa `kms_swrast` can achieve zero-copy software rendering with the help
    of vgem
  - mesa allocates and maps dumb bos for software rendering using vgem
    - it cannot use the kms-only driver because there is no rendernode for the
      kms-only driver
  - it exports dumb bos as dma-bufs and shares them to the compositor
  - the compositor imports the dma-bufs for the kms-only driver
  - `DRM_CAP_DUMB_BUFFER` requires copying
    - mesa `swrast` renders to shmems
    - the compositor copies from shmems to dumb bos
