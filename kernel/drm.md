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

## TTM, using radeon as an example

- the `ttm_bo_device` is initialized in `radeon_ttm_init`
  - the device has `addr_space_mm`, which is used to generate mmap offset of the
    bo for the userspace
  - it initializes three memory types `TTM_PL_SYSTEM`, `TTM_PL_VRAM`, and
    TTM_PL_TT for ttm-internal, vram, and gtt uses.
  - the driver is `radeon_bo_driver`.  `drv->init_mem_type` is called for each
    memory type.  The standard `ttm_bo_manager_func` is used for each of them.
    The standard func uses `drm_mm` to manage the nodes.
- `radeon_gem_create_ioctl`
  - create a `drm_gem_object`, gem bo
  - the gem bo is embedded in a `struct radeon_bo`
  - there is also a `ttm_buffer_object` embedded and it is initialized by
    `ttm_bo_init`
    - the size of the buffer is converted to number of pages needed
    - `ttm_bo_setup_vm` is called to set up the address space in the `drm_mm` so
      that it can be looked up and mapped later in userspace
    - `ttm_bo_validate` is called to validate the ttm bo.  It moves the bo to
      have correct placement.  It calls `ttm_bo_add_ttm` to create the
      associated `ttm_tt`.  A `ttm_tt` is used to manage the physical pages.  It
      uses a `ttm_backend` to be generic.
- `radeon_gem_mmap_ioctl`
  - returns the mmap offset already set up for the bo
- `radeon_mmap`
  - this function is called when the userspace mmap()
  - it calls `ttm_bo_mmap`.  The bo is looked up using the offset.
    `ttm_bo_vm_ops` is installed
  - it replaces the `fault` handler by `radeon_ttm_fault`
  - `ttm_bo_vm_fault` modifies the PTE (page table entry)
    - it calls `fault_reserve_notify` first to make sure the bo can be accessed
    - for vram, it modifies the CPU page table to point to the page in the
      aperture (PCI BAR 0)
    - for others, it uses the page returned by `ttm_tt_get_page`
- when the buffer needs to be moved, `ttm_bo_move_buffer` is called
  - it calls `ttm_bo_mem_space` to reserve a range in the address space of the
    new memory type
  - it then calls `ttm_bo_handle_move_mem` to bind the bo in the new memory type
    and move the data
- when the commands are executed, `radeon_bo_list_validate` is called to
  validate the bo involved
  - 

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
