Kernel DRM
==========

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

## DRM GEM object

- each `drm_gem_object` owns its storage, usually a shmem pointed to by
  `obj->filp`; each shmem has an inode
- each `drm_file` has a `object_idr` for handle-to-gem-obj mapping.  Two
  `drm_file`s can refer to the same gem-obj using different handles.  When the
  last handle is closed, the gem bo is freed.
- each `drm_gem_object` has a unique `name`, allocated from
  `dev->object_name_idr`.  Calling `drm_gem_open_ioctl` twice on the same name
  creates two different handles to same bo.
- each `drm_gem_object` also has a unique `dma_buf`.  Calling
  `drm_gem_prime_fd_to_handle` twice on the same fd returns the same handle.
- `drm_gem_object_funcs`
  - `open`: called when a bo is added to a `drm_file`
  - `close`: called when a bo is removed from a `drm_file`
  - `free`: called when the last reference is gone
  - `export`: called during handle-to-prime to create a `dma_buf`
  - `pin`: called when the exported dma-buf is attached
  - `unpin`: called when the exported dma-buf is detached
  - `get_sg_table`: called when the exported dma-buf is dma-mapped
  - `vmap`: called when the exported dma-buf is vmapped
  - `vunmap`: called when the exported dma-buf is vunmapped
  - `mmap`: called when the bo or the exported dma-buf is mmaped
    - i.e., called from `drm_gem_mmap_obj` or `drm_gem_prime_mmap`
    - it updates vma flags, prot, and ops
  - legacy funcs in `drm_driver`
    - `gem_open_object`
    - `gem_close_object`
    - `gem_free_object_unlocked`
    - `gem_prime_export`
    - `gem_prime_pin`
    - `gem_prime_unpin`
    - `gem_prime_get_sg_table`
    - `gem_prime_vmap`
    - `gem_prime_vunmap`
    - `gem_prime_mmap`
    - no modern equivalent
      - `gem_create_object`
      - `gem_prime_import`

## mmap

- MSM
  - userspace: standard get mmap offset and mmap
    - always `MSM_BO_WC` at allocation time
  - kernel: standard `drm_gem_mmap` followed by `msm_gem_mmap_obj`
    - `msm_gem_fault` pins and faults in pages
  - dma-buf: standard `drm_gem_mmap_obj` followed by `msm_gem_mmap_obj`
- i915
  - userspace: standard get mmap offset and mmap
    - can be `I915_MMAP_OFFSET_WC` or `I915_MMAP_OFFSET_WB` at map time
  - kernel: updates vma flags, prot, ops (`vm_ops_cpu`)
    - `vm_fault_cpu` pins and faults in pages
  - dma-buf: call down `shmem_mmap`
    - `shmem_fault` faults in pages
- amdgpu
  - userspace: standard get mmap offset and mmap
    - `RADEON_FLAG_GTT_WC` may be set or cleared at allocation time
  - kernel: standard `ttm_bo_mmap`
    - `ttm_bo_vm_fault` pins and faults in pages while adjusting prot per-page
  - dma-buf: standard `ttm_bo_mmap`

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

## Memories

- Ideas: GPU
  - GPU has VRAM as the primary storage
  - GPU can use the system memory as the secondary storage
  - As system memory is not physically contiguous, GPU has its own MMU, and the
    page table is called GART.
  - Both VRAM and system memory must be mapped in GART before use.  Either type
    of memory appears to be continuous to GPU with GART.
- Ideas: CPU
  - CPU can map system memory using its MMU
  - CPU can use (a portion of) system memory via AGP aperture
  - CPU can map (a portion of) VRAM via PCI BAR
- Ideas: Sync
  - <http://keithp.com/blogs/gem_update/>
  - VRAM is uncached to CPU
  - System memory mapped through CPU MMU is cached to CPU
  - System memory mapped through AGP aperture is uncached to CPU
    - no meddling with CPU MMU
    - write-combined, which guarantees reasonable streamed write speed
- AGP aperture and GART
  - AGP aperture is a continuous range above TOM (top-of-memory), specified by a
    base and a size (`APBASEHI`+`APBASELO` and `APSIZE`)
  - GART translates a GPU access to an address in the aperture to an physical
    address below TOM.  The table is located at `GARTHI`+`GARTLO` 
- Cards on PCIe use the same concept;  But they are programmed differently
- GPU has a MC (memory controller) too
  - MC is usually programmed to reserve a range for VRAM
  - it reserves another range for AGP aperture when AGP is available
  - (it uses other mechanism to do its own GART when AGP is not available)
- VRAM
  - GPU visibile: yes
  - GPU cached: yes
  - GPU write: very fast
  - GPU read: very fast
  - CPU visible: partial
  - CPU cached: no
  - CPU write: very slow (burst cached)
  - CPU read: extremely slow
- AGP Aperture
  - GPU visible: yes
  - GPU cached: yes
  - GPU write: fast
  - GPU read: fast
  - CPU visible: yes
  - CPU cached: no
  - CPU write: fair (write-combined)
  - CPU read: slow
- System RAM
  - GPU visible: no
  - CPU visible: all
  - CPU write: fast
  - CPU read: fast
- Other than the speed natures of different types of memory, we do not want CPU
  and GPU to access the same buffer due to
  - we do not want to synchronize CPU and GPU (performance hit!)
  - the contents of the buffer may be tiled

## dma-buf

- `drm_prime_handle_to_fd_ioctl`
  - `drm_gem_prime_handle_to_fd` (generic helper)
    - `drm_gem_prime_export` (generic helper)
      - `dma_buf_ops` is `drm_gem_prime_dmabuf_ops`, which uses
      	`drm_gem_object_funcs` of the bo and `drm_driver` of the dev
      - userspace mmap calls driver's `gem_prime_mmap`
    - `i915_gem_prime_export`
      - `i915_dmabuf_ops`
      - userspace mmap calls mmap on the shmem
    - `amdgpu_gem_prime_export`
      - `amdgpu_dmabuf_ops`
      - userspace mmap calls driver's `gem_prime_mmap`, which calls
      	`drm_gem_ttm_mmap`

## Radeon and DMA

- <http://www.botchco.com/agd5f/?p=50>
- A radeon may be on AGP, PCIE, or PCI
  - if the radeon is on AGP and the init failed, it is re-initialized as PCIE or
    PCI
- The MC is programmed to reserve
  - `R_000148_MC_FB_LOCATION` for VRAM
  - `R_00014C_MC_AGP_LOCATION` for AGP when available; the offset into the range
    is added by `R_000170_AGP_BASE` to produce an address in the AGP aperture
- When AGP is not available, the GART is programmed in a way such that an access
  to `RADEON_AIC_LO_ADDR`+`RADEON_AIC_HI_ADDR` is translated using the GART at
  `RADEON_AIC_PT_BASE`.  Similar to AGP.
  - `RADEON_AIC_PT_BASE` is set to the DMA address of the GART table
  - each entry in the gart table is a 32-bit address
  - an entry maps a GPU page number to the DMA address
- In `radeon_check_arguments`, vram is unlimited; gart (`rdev->mc.gtt_size`) is
  limited to 512MB
  - the gart size will be later set to the AGP aperture size if on AGP
- `rdev->mc.mc_vram_size` is set to the size of vram
  - it may be stolen system memory if on IGP
- example
  - VRAM: 1024M 0x0000000000000000 - 0x000000003FFFFFFF (1024M used)
  - GTT: 512M 0x0000000040000000 - 0x000000005FFFFFFF 
  - Detected VRAM RAM=1024M, BAR=256M
- MC
  - VRAM
    - `aper_base` the base address of PCI BAR 0 (mapped VRAM for CPU)
    - `aper_size` the size of PCI BAR 0
    - `visible_vram_size` the size of vram visible to CPU
    - `real_vram_size` the physical size of vram
    - `mc_vram_size` the range to reserve in MC for vram.  Could be larger than
      `real_vram_size`
  - GART
    - `agp_base` the base address of the AGP apertise (for use by GPU)
    - `gtt_size` is set to the AGP apertise size
  - `vram_start` is the start address of VRAM in GPU's view
    - it is 0 for some chipsets; is `aper_base` for others; but it doesn't
      matter
  - `gtt_start` is the start address of GTT in GPU's view
    - it is after `vram_end` for non-AGP; is `agp_base` for AGP
- FB
  - for CPU, vram starts at `mc.aper_base`
  - Say fb bo is at `mc.vram_start + offset`.  It is accesible from CPU at
    `mc.aper_base + offset`, which is fb `smem_start`
- there are `radeon_device` and `drm_radeon_private`
- `resource_copy_region`
- `clear`
- buffer transfer
  - `transfer_inline_write` maps a buffer with DISCARD, memcpy, and unmaps
  - other ops are the trivial
- texture transfer
  - simple except whe the resource is tiled
  - a linear staging texture is created; if for reading, `resource_copy_region`
    is called to copy the tiled texture into the linear texture; if for writing,
    `resource_copy_region` is also called
- `ttm_bo_reserve` is a lockless lock of the bo

## Old

DRI server
- drmGetInterruptFromBusID to get irq number
- drmCtlInstHandler to enable irq (request_irq)
- drmAddBufs to add DMA fixed-size buffers (old..)

DRI client (radeon)
- drmMapBufs to map DMA buffers.  Should drmDMA befure use.
- drmCommandWrite DRM_RADEON_CMDBUF to send buffer
- drmCommandNone DRM_RADEON_CP_IDLE to busy wait for idle
- alternatively, drmCommandWriteRead DRM_RADEON_IRQ_EMIT and
  drmCommandWrite DRM_RADEON_IRQ_WAIT to wait for idle irq
- the latter is used when the hw supports irq and the client
  wants to use it to throttle.
  (is it possible that hw supports irq but does not enable it?)

DRI client (intel)
- newer intel does not use DMA buffer and it never busy waits
- it allocs a buffer from fake bufmgr which uses the area given by SAREA 
- it waits for irq
- for GEM, it uses GEM buffer and waits for GTT access.
  The commands are stored in a malloc()'ed buffer.
  The buffer is submitted just before flush and pwrite to the gem
- older intel (i810) is similar to radeon

Glossary
- CRTCs: the chips controlling connectors (each CRTC has one or more connectors)
- Connector: LVDS, VGA, DVI, etc.
- Screen: may span over multiple monitors, thus multiple CRTCs and connectors
