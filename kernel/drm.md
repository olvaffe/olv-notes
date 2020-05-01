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

## KMS Datapaths

* framebuffers feed data into planes, 1:1
* planes feed data into crtc, N:1
* crtc feed data into encoders, 1:N
  * encoders should be an implementation detail, but they are unfortunately a
    part of the public APIs
* encoders feed data into connectors, 1:1
  * there might be a chain of brdiges between the encoder and the connector
* connectors feed data into the physical displays
* in an atomic update, we can change all datapaths, or just a datapth, or just
  a sub-datapath.  E.g., we can update just one `fb->plane->crtc` subpath for
  a pageflip
* to change a datapath end-to-end, it usually requires
  * disable output
    * `->atomic_disable` the bridge chain, starting from the tail (which
      connects to the connector)
    * `->atomic_disable` the encoder
    * `->atomic_post_disable` the bridge chain, starting from the head (which
      connects to the encoder)
    * `->atomic_disable` the crtc
  * change mode
    * `->mode_set_nofb` the crtc
    * `->atomic_mode_set` the encoder
    * `->mode_set` the bridge chain, starting from the head
  * update plane
    * `->atomic_begin` the crtc
    * `->atomic_update` (or `->atomic_disable`) the plane
    * `->atomic_flush` the crtc
  * enable output
    * `->atomic_enable` the crtc
    * `->atomic_pre_enable` the bridge chain, starting from the tail
    * `->atomic_enable` the encoder
    * `->atomic_enable` the bridge chain, starting from the head

## Atomic Modesetting

* `DRM_IOCTL_MODE_ATOMIC`
* we will have many components (crtcs, planes, connectors) to lock down, and
  we don't want a global lock, it necessitates `ww_mutex`
* we colloect all information needed for the atomic update in `drm_atomic_state`
  * for each component included in this atomic update, we lock it down, we
    duplicate its state, and we modify only the duplicated state first
  * a crtc or a connnector might have an out fence.  It is a fence that the
    atomic update must signal after the update completes (usually on next
    vsync)
  * a plane might have an in fence.  It is a fence that the atomic update must
    wait before updating the plane
* with `drm_atomic_state` ready, the driver can look at it and decide whether
  it is doable
* when there are plane out fences or the page flip event is requested
  * we create a `drm_pending_vblank_event` for each crtc interests in it
  * the event will be delivered when the atomic update completes and signals
    the plane out fences
* now is time to call into the driver `->atomic_commit` hook.  If the HW is
  very standard, the driver can use `drm_atomic_helper_commit`
  * `drm_atomic_helper_setup_commit` to set up primitives that help track the
    progress of this commit
    * also check the progress of the previous commit; wait for it if needed
  * `drm_atomic_helper_prepare_planes` to prepare fbs
    * pin the pages of the new fb
    * collect fences for the old (before we can unpin) and new fbs that it
      will wait
  * schedule commit
    * SW should wait for in fences or implicit/internal in fences to signal
      before updating registers
    * modeset registers can be single-buffered or double-buffered.  With
      double-buffered ones, they can be set at any time and will be latched on
      vsyncs; with single-buffered ones, SW should set them on next vsync
    * we don't want atomic update to be blocked so we schedule the commit to
      a workqueue
* finally, return the out fence fds to the userspace
* in the workqueue that does the blocking commit, if the HW is very
  standard, it can
  * `drm_atomic_helper_wait_for_fences` to wait for all fences to signal
  * `drm_atomic_helper_wait_for_dependencies` to wait for preceding commits
  * `drm_atomic_helper_commit_tail`
    * `drm_atomic_helper_commit_modeset_disables`
      * `->atomic_disable` each crtc
    * `drm_atomic_helper_commit_planes` to update the registers
      * `->atomic_begin` each crtc
      * `->atomic_update` each plane
      * `->atomic_flush` each crtc
    * `drm_atomic_helper_commit_modeset_enables`
      * `->atomic_enable` each crtc
    * `drm_atomic_helper_commit_hw_done` to update the progress
    * `drm_atomic_helper_wait_for_vblanks` to wait until next vsync
    * `drm_atomic_helper_cleanup_planes` to clean up after the commit
  * `drm_atomic_helper_commit_cleanup_done` to update pregress
  * no failure allowed
* In userspace API
  * `DRM_MODE_ATOMIC_NONBLOCK` means `DRM_IOCTL_MODE_ATOMIC` should be
    non-blocking
  * `DRM_MODE_PAGE_FLIP_ASYNC` means tearing is fine; the atomic commit does
    not need to happen on vsyncs

## Atomic Modesetting as the Driver Mechanism

* `DRM_IOCTL_MODE_PAGE_FLIP` can be implemented with the generic
  `drm_atomic_helper_page_flip`
  * it does a non-blocking atomic commit to udpate a `fb->plane->crtc`
    sub-datapth
* `DRM_IOCTL_MODE_DIRTYFB` can be implemented with the genric
  `drm_atomic_helper_dirtyfb`
  * it does a blocking atomic commit to udpate the `FB_DAMAGE_CLIPS` property
    of a plane

## Memories

* Ideas: GPU
  * GPU has VRAM as the primary storage
  * GPU can use the system memory as the secondary storage
  * As system memory is not physically contiguous, GPU has its own MMU, and the
    page table is called GART.
  * Both VRAM and system memory must be mapped in GART before use.  Either type
    of memory appears to be continuous to GPU with GART.
* Ideas: CPU
  * CPU can map system memory using its MMU
  * CPU can use (a portion of) system memory via AGP aperture
  * CPU can map (a portion of) VRAM via PCI BAR
* Ideas: Sync
  * <http://keithp.com/blogs/gem_update/>
  * VRAM is uncached to CPU
  * System memory mapped through CPU MMU is cached to CPU
  * System memory mapped through AGP aperture is uncached to CPU
    * no meddling with CPU MMU
    * write-combined, which guarantees reasonable streamed write speed
* AGP aperture and GART
  * AGP aperture is a continuous range above TOM (top-of-memory), specified by a
    base and a size (`APBASEHI`+`APBASELO` and `APSIZE`)
  * GART translates a GPU access to an address in the aperture to an physical
    address below TOM.  The table is located at `GARTHI`+`GARTLO` 
* Cards on PCIe use the same concept;  But they are programmed differently
* GPU has a MC (memory controller) too
  * MC is usually programmed to reserve a range for VRAM
  * it reserves another range for AGP aperture when AGP is available
  * (it uses other mechanism to do its own GART when AGP is not available)
* VRAM
  * GPU visibile: yes
  * GPU cached: yes
  * GPU write: very fast
  * GPU read: very fast
  * CPU visible: partial
  * CPU cached: no
  * CPU write: very slow (burst cached)
  * CPU read: extremely slow
* AGP Aperture
  * GPU visible: yes
  * GPU cached: yes
  * GPU write: fast
  * GPU read: fast
  * CPU visible: yes
  * CPU cached: no
  * CPU write: fair (write-combined)
  * CPU read: slow
* System RAM
  * GPU visible: no
  * CPU visible: all
  * CPU write: fast
  * CPU read: fast
* Other than the speed natures of different types of memory, we do not want CPU
  and GPU to access the same buffer due to
  * we do not want to synchronize CPU and GPU (performance hit!)
  * the contents of the buffer may be tiled

## TTM, using radeon as an example

* the `ttm_bo_device` is initialized in `radeon_ttm_init`
  * the device has `addr_space_mm`, which is used to generate mmap offset of the
    bo for the userspace
  * it initializes three memory types `TTM_PL_SYSTEM`, `TTM_PL_VRAM`, and
    TTM_PL_TT for ttm-internal, vram, and gtt uses.
  * the driver is `radeon_bo_driver`.  `drv->init_mem_type` is called for each
    memory type.  The standard `ttm_bo_manager_func` is used for each of them.
    The standard func uses `drm_mm` to manage the nodes.
* `radeon_gem_create_ioctl`
  * create a `drm_gem_object`, gem bo
  * the gem bo is embedded in a `struct radeon_bo`
  * there is also a `ttm_buffer_object` embedded and it is initialized by
    `ttm_bo_init`
    * the size of the buffer is converted to number of pages needed
    * `ttm_bo_setup_vm` is called to set up the address space in the `drm_mm` so
      that it can be looked up and mapped later in userspace
    * `ttm_bo_validate` is called to validate the ttm bo.  It moves the bo to
      have correct placement.  It calls `ttm_bo_add_ttm` to create the
      associated `ttm_tt`.  A `ttm_tt` is used to manage the physical pages.  It
      uses a `ttm_backend` to be generic.
* `radeon_gem_mmap_ioctl`
  * returns the mmap offset already set up for the bo
* `radeon_mmap`
  * this function is called when the userspace mmap()
  * it calls `ttm_bo_mmap`.  The bo is looked up using the offset.
    `ttm_bo_vm_ops` is installed
  * it replaces the `fault` handler by `radeon_ttm_fault`
  * `ttm_bo_vm_fault` modifies the PTE (page table entry)
    * it calls `fault_reserve_notify` first to make sure the bo can be accessed
    * for vram, it modifies the CPU page table to point to the page in the
      aperture (PCI BAR 0)
    * for others, it uses the page returned by `ttm_tt_get_page`
* when the buffer needs to be moved, `ttm_bo_move_buffer` is called
  * it calls `ttm_bo_mem_space` to reserve a range in the address space of the
    new memory type
  * it then calls `ttm_bo_handle_move_mem` to bind the bo in the new memory type
    and move the data
* when the commands are executed, `radeon_bo_list_validate` is called to
  validate the bo involved
  * 

## Radeon and DMA

* <http://www.botchco.com/agd5f/?p=50>
* A radeon may be on AGP, PCIE, or PCI
  * if the radeon is on AGP and the init failed, it is re-initialized as PCIE or
    PCI
* The MC is programmed to reserve
  * `R_000148_MC_FB_LOCATION` for VRAM
  * `R_00014C_MC_AGP_LOCATION` for AGP when available; the offset into the range
    is added by `R_000170_AGP_BASE` to produce an address in the AGP aperture
* When AGP is not available, the GART is programmed in a way such that an access
  to `RADEON_AIC_LO_ADDR`+`RADEON_AIC_HI_ADDR` is translated using the GART at
  `RADEON_AIC_PT_BASE`.  Similar to AGP.
  * `RADEON_AIC_PT_BASE` is set to the DMA address of the GART table
  * each entry in the gart table is a 32-bit address
  * an entry maps a GPU page number to the DMA address
* In `radeon_check_arguments`, vram is unlimited; gart (`rdev->mc.gtt_size`) is
  limited to 512MB
  * the gart size will be later set to the AGP aperture size if on AGP
* `rdev->mc.mc_vram_size` is set to the size of vram
  * it may be stolen system memory if on IGP
* example
  * VRAM: 1024M 0x0000000000000000 - 0x000000003FFFFFFF (1024M used)
  * GTT: 512M 0x0000000040000000 - 0x000000005FFFFFFF 
  * Detected VRAM RAM=1024M, BAR=256M
* MC
  * VRAM
    * `aper_base` the base address of PCI BAR 0 (mapped VRAM for CPU)
    * `aper_size` the size of PCI BAR 0
    * `visible_vram_size` the size of vram visible to CPU
    * `real_vram_size` the physical size of vram
    * `mc_vram_size` the range to reserve in MC for vram.  Could be larger than
      `real_vram_size`
  * GART
    * `agp_base` the base address of the AGP apertise (for use by GPU)
    * `gtt_size` is set to the AGP apertise size
  * `vram_start` is the start address of VRAM in GPU's view
    * it is 0 for some chipsets; is `aper_base` for others; but it doesn't
      matter
  * `gtt_start` is the start address of GTT in GPU's view
    * it is after `vram_end` for non-AGP; is `agp_base` for AGP
* FB
  * for CPU, vram starts at `mc.aper_base`
  * Say fb bo is at `mc.vram_start + offset`.  It is accesible from CPU at
    `mc.aper_base + offset`, which is fb `smem_start`
* there are `radeon_device` and `drm_radeon_private`
* `resource_copy_region`
* `clear`
* buffer transfer
  * `transfer_inline_write` maps a buffer with DISCARD, memcpy, and unmaps
  * other ops are the trivial
* texture transfer
  * simple except whe the resource is tiled
  * a linear staging texture is created; if for reading, `resource_copy_region`
    is called to copy the tiled texture into the linear texture; if for writing,
    `resource_copy_region` is also called
* `ttm_bo_reserve` is a lockless lock of the bo


## Events

* Kernel sends three types of events to the userspace
  * `DRM_EVENT_VBLANK`, in response to `DRM_IOCTL_WAIT_VBLANK` with
    `_DRM_VBLANK_EVENT` bit
  * `DRM_EVENT_FLIP_COMPLETE`, in response to `DRM_IOCTL_MODE_PAGE_FLIP` or
    `DRM_IOCTL_MODE_ATOMIC`, with `DRM_MODE_PAGE_FLIP_EVENT` bit
  * `DRM_EVENT_CRTC_SEQUENCE`, in response to `DRM_IOCTL_CRTC_QUEUE_SEQUENCE`
* `drmWaitVBlank`
  * there is a counter for the number of vblanks since the system booted
  * `_DRM_VBLANK_ABSOLUTE` waits until the counter matches the specified number 
  * `_DRM_VBLANK_RELATIVE:` waits for the number of vblanks specified
  * `_DRM_VBLANK_SECONDARY` selects whether crtc 0 or 1 should be used
  * `_DRM_VBLANK_NEXTONMISS` waits for next vblank if the specified one is
    missed
  * Unless `_DRM_VBLANK_EVENT` is set, the caller waits for the specified
    vblank.  When it happens, the current time of day and the current vblank is
    returned.
  * When `_DRM_VBLANK_EVENT` is set, instead of blocking, the call returns
    immediately.
    * a pending event is added to `dev->vblank_event_list`
    * driver calls `drm_handle_vblank` in its IRQ handler to move the event to
      `file_priv->event_list`
    * read() on the fd returns the event
* `drmModePageFlip`
  * see d91d8a3f88059d93e34ac70d059153ec69a9ffc7
  * is handled by driver, such as `intel_crtc_page_flip`
  * the pending event is NULL unless `DRM_MODE_PAGE_FLIP_EVENT` is set
  * the current fb (`work->old_fb_obj`) is remembered
  * the new fb (`work->pending_flip_obj`) is pinned and fenced, and
    `i915_gem_object_flush_write_domain` is called
    * note that a flush in kernel does not mean to submit the instructions to
      GPU.  It adds a `MI_FLUSH` instruction so that the GPU would flush its
      caches.
  * a `MI_DISPLAY_FLIP` instruction is emitted
    * after flush, so that all renderings are guaranteed to be done and hit the
      memory by the time flip happens
  * before the flip is done, `obj->pending_flip` is non-zero.  If the obj is
    used by subsequent rendering before the flip happens,
    `i915_gem_wait_for_pending_flip` is called to wait for the flip.
  * after the flip is done, as notified by IRQ, `intel_finish_page_flip` is
    called.  if a pending event was given, it is added to
    `file_priv->event_list`.  The old fb is unpinned.
* `drmModeSetCrtc`
  * `drm_mode_setcrtc -> drm_crtc_helper_set_config -> intel_pipe_set_base`
  * the new fb is pinned and fenced
  * `i915_gem_object_set_to_display_plane` emits flush and waits, and finally
    move the obj to GTT
  * CRTC is programmed immediately

DRI server
* drmGetInterruptFromBusID to get irq number
* drmCtlInstHandler to enable irq (request_irq)
* drmAddBufs to add DMA fixed-size buffers (old..)

DRI client (radeon)
* drmMapBufs to map DMA buffers.  Should drmDMA befure use.
* drmCommandWrite DRM_RADEON_CMDBUF to send buffer
* drmCommandNone DRM_RADEON_CP_IDLE to busy wait for idle
* alternatively, drmCommandWriteRead DRM_RADEON_IRQ_EMIT and
  drmCommandWrite DRM_RADEON_IRQ_WAIT to wait for idle irq
* the latter is used when the hw supports irq and the client
  wants to use it to throttle.
  (is it possible that hw supports irq but does not enable it?)

DRI client (intel)
* newer intel does not use DMA buffer and it never busy waits
* it allocs a buffer from fake bufmgr which uses the area given by SAREA 
* it waits for irq
* for GEM, it uses GEM buffer and waits for GTT access.
  The commands are stored in a malloc()'ed buffer.
  The buffer is submitted just before flush and pwrite to the gem
* older intel (i810) is similar to radeon

Glossary
* CRTCs: the chips controlling connectors (each CRTC has one or more connectors)
* Connector: LVDS, VGA, DVI, etc.
* Screen: may span over multiple monitors, thus multiple CRTCs and connectors

aperture
* i915_probe_agp returns aperture and preallocated sizes.
  the whole memory is called aperture.  the beginning some memory is stolen.
  it consists of gtt and preallocated memories.
* two drm_mm is created to manage preallocated memory and the reset of the aperture.
* drm_mm does not touch real memory, only offsets and sizes, which makes it generic

Memory Domains
* CPU: cpu addr -> mmu -> system ram
* GTT: cpu addr -> agp -> gpu addr -> gart -> system ram
* GPU: ~(CPU | GTT)
* GTT space is limited, comparing to CPU space
* there is always one write domain
* there may be multiple read domains

drm_gem_object
* DRM_I915_GEM_CREATE creates drm_gem_object through drm_gem_object_alloc.
* the object is shmem-based and a handle is added
* i915_gem_object_get_page_list checks the backing memory pointer and
  constructs a page_list for the physical pages of the memory.
* i915_gem_object_bind_to_gtt finds a free space in aperture and binds the
  backing pages of the object there.
* pin is almost equal to bind gtt.
* active object cannot be unbound.  bound or pinned object may be unmanaged, i.e. not on any list.
  when a request is retired, it is moved either to flushing or inactive list.
* mmap gtt creates a fake offset to allow userspace to mmap this object in
  aperture.  It binds the object.  The domain should be GTT
* mmap (non-gtt) maps the backing shmem of the object.  The difference from gtt
  mapping is that accessing gtt mapped region goes through agp and it makes
  linearly accessing tiled buffer possible.
* intel bufmgr drm_intel_gem_bo_map calls DRM_IOCTL_I915_GEM_MMAP to
  mmap the backing shmem and move the object to CPU domain.
  It is for swrast.
* glamo for example, bind/unbind moves object to/from vram.  set_domain flushes
  cpu caches or glamo rendering.
* gem on aperture may be tiled.  to make gtt access seems linear, fence regs
  must be set up.

lifetime
* an object is active if it is on active or flushing list.  active object
  cannot be unbound.
* when a pinned object is done, it becomes unmanaged because it cannot be
  evicted.
* an object on active list with write_domain will be moved to flushing list
  when done.  one without write_domain will be moved to inactive list.
* i915_add_request creates a seqno and installs an interrupt.  It removes
  the write domain of objects on flushing list with the specified flushing domains
  and moves them to active list.
* i915_wait_request waits until the seqno passes the given one.
* i915_gem_retire_request retires the first object(s) matching req->seqno by
  moving them to flushing or inactive list depending on their write_domain.
* i915_gem_retire_requests retires all passed requests.
* i915_gem_flush invalidates and/or flushes the given domains by putting the
  commands on the ring.
* i915_gem_evict_something is called when there is no aperture.  If there is
  inactive object, it unbinds the object.  Otherwise, it waits for a request.
  to finish, hoping it will make some room.  If there is no request, the first
  object on flushing list is flushed.  A requets is made and waited in that case.

mmap
* when /dev/drm/card? is mmaped, drm_gem_mmap is called
* if the region has a fake offset (GTT mapped), gem_vm_ops is installed
* otherwise, drm_mmap is called.
* if there is no offset and no agp, drm_mmap_dma is called and drm_vm_dma_ops is installed.
* otherwise, REGISTERS/FRAME_BUFFER/... is mapped
* it is used by drmAddMap and drmMap

PIO
* when pread, i915_gem_object_set_cpu_read_domain_range is called to move
  intetested range to CPU domain (not entirely valid).
  For write domain, i915_gem_object_flush_gpu_write_domain and
  i915_gem_object_wait_rendering are called so that there is no
  unexpcted writes.  After then, drm_clflush_pages is called to invalidate
  CPU caches.  There seems to be a missing drm_agp_chipset_flush which should
  be called after clflush??
*

/dev/dri/cardX
* Given a pci device and a type (always known by its driver, see drm_get_dev), 
  a new minor_id and a drm_minor is associated.  An dev node is created with
  the minor.
* every cardX is a drm_minor; every open creates a drm_file as file private.
  A drm_master for the drm_minor is created if none exists, and the drm_file
  becomes a master and is automatically authenticated.
* open a cardX -> drm_stub_open -> install and use driver fops, which usually
  has drm_open as its .open

modeset
* the drm_device contains mode_config, which is a drm_mode_config
* drm_mode_config has, among others, a list of crtcs, a list of connectors, and a list of encoders
* connectors have an encoder and modes, which are the modes the connected display supports
  crtc has a current mode and a desired mode
  The relation is (my guess): card -> crtcs -> routing encoders -> connectors (connector has no encoder if not routed)
* connector has encoder id; encoder has crtc id; crtc has buffer id
* after deciding the desired mode, userspace creates a bo, ADDFB, and SETCRTC.
  ADDFB creates intel_framebuffer around the bo.
  SETCRTC sets the specified crtc.  This includes the scan-out fb and the mode of the crtc,
	and the connectors connected to it.  As for the latter, it might involve move connectors
	from other crtc to this one or disconnect some connectors from this crtc
* i915 resources on eeepc

        Resources
        
        count_connectors : 3
        count_encoders   : 3
        count_crtcs      : 2
        count_fbs        : 0
        
        Connector: 1-1
                id             : 5
                encoder id     : 0
                conn           : unknown
                size           : 0x0 (mm)
                count_modes    : 0
                count_props    : 2
                props          : 1 2
                count_encoders : 1
                encoders       : 6
        Connector: 7-1
                id             : 7
                encoder id     : 8
                conn           : connected
                size           : 0x0 (mm)
                count_modes    : 1
                count_props    : 2
                props          : 1 2
                count_encoders : 1
                encoders       : 8
        Mode: "1024x600" 1024x600 60
        Connector: 6-1
                id             : 10
                encoder id     : 0
                conn           : disconnected
                size           : 0x0 (mm)
                count_modes    : 0
                count_props    : 7
                props          : 1 2 18 14 16 15 17
                count_encoders : 1
                encoders       : 11
        
        Encoder
                id     :6
                crtc_id   :0
                type   :1
                possible_crtcs  :3
                possible_clones :1
        Encoder
                id     :8
                crtc_id   :4
                type   :3
                possible_crtcs  :2
                possible_clones :2
        Encoder
                id     :11
                crtc_id   :0
                type   :4
                possible_crtcs  :3
                possible_clones :4
        
        Crtc
                id             : 3
                x              : 0
                y              : 0
                width          : 0
                height         : 0
                mode           : 0x804b4c4
                gamma size     : 256
        Crtc
                id             : 4
                x              : 0
                y              : 0
                width          : 0
                height         : 0
                mode           : 0x804b4c4
                gamma size     : 256


## i915 modeset

* `intel_modeset_init`
* Depending on the chipset, 1 or 2 pipes (`intel_crtc`) are created.
* Depending on the chipset again, some of `intel_crt_init`, `intel_lvds_init`,
  `intel_sdvo_init`, `intel_hdmi_init`, `intel_dvo_init`, `intel_tv_init` are
  called.  Each function creates a `intel_output`, which consists of
  * a `drm_connector` of types like `DRM_MODE_CONNECTOR_VGA`
  * a `drm_encoder` of types like `DRM_MODE_ENCODER_DAC`
  * `drm_mode_connector_attach_encoder` is called to setthe connector's `encoder_ids`
* each `intel_output`'s encoder is set hardcoded values, depending on output
  type
  * `possible_crtcs`: only the first two bits are set (which pipes supported?)
  * `possible_clones`: bitmaps of supported connectors (connectors are ordered?)
* LVDS is always the second pipe?
* `drmModeSetCrtc` corresponds to `drm_crtc_helper_set_config` in kernel.  It
  finds a best encoder for each given connector, matches given connectors'
  best encoders with the given crtc (and disconnects those not given).
  `drm_crtc_helper_set_mode` and `drm_helper_disable_unused_functions` is
  called.

## PREAD/PWRITE

* pread/pwrite are reading/write from _CPU_ perspective, according to
  `i915_gem_pwrite_ioctl`.
* `i915_gem_object_get_pages` gets and stores the pages of the backing shmem in
  `struct drm_i915_gem_object`'s `pages`.  `i915_gem_object_do_bit_17_swizzle`
  is called if the object is tiled.
* `i915_gem_object_bind_to_gtt` finds an unused region from
  `dev_priv->mm.gtt_space` and it becomes the `gtt_space` of the object.  It
  thens binds object's pages to its `gtt_space` using `drm_agp_bind_pages`.
* When `i915_gem_mmap_gtt_ioctl` is called, the object's `mmap_offset` is
  determined, and it is bound to gtt.
* Now the user can gtt mmap the object using its `mmap_offset`.  The fault
  handler is `i915_gem_fault`.  It fauls in the pages based on `dev->agp`, with
  the fence reg properly set (when tiled).
* When an object is bound to gtt, it is put in the `inactive_list`.
  * It means it is inactive to the GPU, and can be evicted.
* When an object is pinned, it is removed from the `inactive_list`.
  * It is so that the object is never considered for eviction.
* When evicting something, it looks for inactive object first.  It calls
  `i915_gem_object_unbind` on the first object it finds.
  * If the object is pinned, unbind fails.
  * `agp_unbind_memory` is called to make room in gatt
  * `i915_gem_object_put_pages` is called to return the pages storing the
    contents of the shmem.
  * `drm_mm_put_block` is called to give the region back to `gtt_space`
  * the object is removed from any list

## Tiling

* Tiled address of `Vol_1b_G45_core.pdf`.
* Channel XOR
  * For CPU, bit6 affects none, bit 11 or bit 17
  * For GPU, bit6 affects 
    * none, if no tiling
    * bit 9, if y tiling
    * bit 9 and bit 10, if x tiling
  * When we say bitX affects bitY, it means the bitY of an physical address is
    XORed with bitX of the same address.

## MMAP

* `struct drm_gem_object` is backed by shmem created by `shmem_file_setup`.
  * It has `shmem_file_operations` as its `f_op`.
* `i915_gem_mmap_ioctl` calls `do_mmap` on the underlying shmem.
  * a `struct vm_area_struct` is created
  * `file->f_op->mmap` is called on the vma.  It sets `vma->vm_ops` to
    `shmem_vm_ops`.
  * Whenever the user acceses the memory, the pages are faulted in through
    `shmem_fault`.
* `i915_gem_mmap_gtt_ioctl` binds the backing pages to agp through
  `drm_agp_bind_pages`.
  * When the user mmap it, an vma with `i915_gem_vm_ops` is created.
  * the fault function sets up the pte to access the faulted in page from agp.

## `agp_bind_memory`

* It takes a `struct agp_memory` and a `off_t`.
  * The former gives the gart addresses of the pages to be bound
  * The latter gives the offset into the AGP aperture where the map should base
* It calls `bridge->driver->insert_memory`.
* In case of `agp_generic_insert_memory`,
  * It updates the gatt table.
  * gatt table is an array of physical addresses, aligned on page boundaries
  * when accessing through agp aperture, if the address falls in gatt entry i,
    it acceses the memory given by the physical address of entry i.
* E.g. `i915_gem_fault`
  * It knows `page_offset`, the page in the object the user wants to access
  * It knows `gtt_offset`, the first entry of the object in the gatt table
  * It knows `agp->base`, the base of the agp aperture
  * The logical address the user should access is
    `agp->base + gtt_offset + (page_offset << PAGE_SHIFT)`
  * The physical adress the user will access is decided by gatt entry
    `(gtt_offset >> PAGE_SHIFT) + page_offset`.

## Object Status

* Object has 3 status, inactive, active, and flushing
* objects are moved to active list when they are used by a request
* active objects are moved to other lists when the request is retired
  * objects being read are moved to inactive list
  * objects being written are moved to flushing list
* flushing objects will become active before it becomes inactive
  * when a request is added, the domains its commands flush are provided.
    They are used to move flushing objects to active objects, and those objects
    will be moved to inactive list when the request is retired.
* Objects on inactive list might be evicted at any time
  * Pinning an object removes it from the inactive list
  * Unpinning an object adds it to the inactive list if it is inactive

## domains

* a domain indicates two things
  * an object is readable/writable from that domain.
  * an object is assumed to have read/written from/to that domain.
  * the latter decides whether flushing is needed in some places.
* In `i915_gem_execbuffer`, relocs are applied such that objects have pending
  domain changes in `pending_read_domains` and `pending_write_domain`.
  * `i915_gem_object_set_to_gpu_domain`, which is only called in that function,
    reuses those fields too.
  * `i915_gem_flush` is called to add commands to accomplish the pending
    changes.  `i915_add_request` is also called to move flushing objects to
    active list.
  * pending domains are copied to current domains.
* `i915_gem_flush` puts command to the ring buffer to invalidate/flush gpu
  domains
  * A domain might read its cached data.  If we are going to read from that
    domain and we know that some other domains have modified the data, the
    domain should be invalidated.
  * A domain might write to its cache.  If we are or know other domains are
    going to read the data, the domain should be flushed first.
* A flushed write domain will not be a write domain
  * it flushes the written data
  * _AND_ it unsets the flag in `write_domain`.
  * this is because a domain is flushed to be read by others.  We don't want it
    to be modified in-between.
* Setting to a domain involves
  * make sure there is no `write_domain` by flushing current write domains
  * make sure the new domain is readable/writable
  * update the domain flags
* `i915_gem_idle` waits the hw to be idle
  * it is called when switching away from current vt or suspending

## execbuffer

* The struct passed to `i915_gem_execbuffer` is basically an array of
  `struct drm_i915_gem_exec_object`.
  * It gives the object in question and its associated relocs
  * Usually, all but the last exec objects have no relocs
  * The last exec object describes the batchbuffer.  Its relocs' targets are
    given in that same array.
* First, the exec object array is copied into the kernel space
  * `exec_list` is the array
  * `relocs` is the relocs collected from each exec object
  * `object_list` is the gem objects corresponding to the exec objects
* The DRM device is locked at this point.
* Each exec object is pinned and relocated by `i915_gem_object_pin_and_relocate`
  * It pins the given object.
  * The object's buffer is patched by the relocs.
    * reloc's offset gives where in the buffer needs to be patched
    * reloc's delta is added by target's `gtt_offset` to form the final value
    * reloc's domains is the pending domains of the target
  * Usually, only the last exec object has relocs.
* Each object is `i915_gem_object_set_to_gpu_domain`.
* It puts commands to move objects to desired domains.
* It asks the GPU to execute by calling `i915_dispatch_gem_execbuffer`.
* `i915_retire_commands` is called to insert flush command to the end
* `i915_add_request` is called
  * to add commands to write a monotonically increasing seqno into
    `I915_GEM_HWS_INDEX` and to interrupt when executed.
  * to record the seqno and add it to `request_list`.
  * to move objects on the flushing list to active list, if they are known to be
    flushed by the commands of this request.
  * to schedule `i915_gem_retire_work_handler` to timeout request.
* All objects are moved to `active_list`.
  * with a seqno denoting which request they are active for.
* All objects are unpinned.
* ...
* The DRM device is unlocked.
* Some of the per-device or per-object variables are used only within the lock.
* later when `i915_gem_retire_requests` is called (1HZ after the request is
  added), objects are moved to inactive list or flushing list.
  * it calls `i915_gem_retire_request` to require requests
  * objects that are modified (has write domain) are moved to flushing list
  * the others are moved to inactive

## dumb

- `DRM_IOCTL_MODE_CREATE_DUMB`
  - in: width, height, bpp
  - out: handle, pitch, size
- `DRM_IOCTL_MODE_ADDFB`
  - in: width, height, bpp, pitch, depth
- `DRM_IOCTL_MODE_MAP_DUMB` return an fake offset for mmap
  - for software rendering
- `DRM_CAP_DUMB_BUFFER` checks if the driver supports dumb
- `DRM_CAP_DUMB_PREFERRED_DEPTH` returns the preferred depth
  - static, usually 24 (because alpha is ignored)
  - 0 on virtio-gpu
- `DRM_CAP_DUMB_PREFER_SHADOW` returns whether shadow is preferred
  - static, usually true (because dumb might be in vram and/or uncached)
  - false on virtio-gpu
- xorg-modesetting without gbm does
  - when preferred depth is 8 or 16, depth=16, bpp=16, kbpp=0
  - otherwise,
    - depth=24, bpp=32, kbpp=0 if bpp=32 dumb and fb is supported
    - depth=24, bpp=32, kbpp=24 and forces bpp=32 shadow
- i915 assumes the dumb bo format to be `DRM_FORMAT_C8`, `DRM_FORMAT_RGB565`,
  or `DRM_FORMAT_XRGB8888` depending on bpp

## KMS object properties

- `DRM_IOCTL_MODE_OBJ_GETPROPERTIES` can get all properties of a KMS object
  - in: object id and object type `DRM_MODE_OBJECT_*`
  - out: an array of (32-bit prop id, 64-bit prop val) pairs
  - each prop id seems to be globally unique (see below)
- `DRM_IOCTL_MODE_GETPROPERTY` can get info about a prop id
  - in: prop id
  - out: name and values
    - name is well-defined and is a part of the API
    - values depends on the flags
      - for `DRM_MODE_PROP_RANGE`, values are 64-bit pairs
      - for `DRM_MODE_PROP_ENUM`, values are an array of
        (well-defined enum name, 64-bit enum id)
      - for `DRM_MODE_PROP_BITMASK` is similar to `DRM_MODE_PROP_ENUM`
      - for `DRMO_MODE_PROP_BLOB`, values are an array of 32-bit blob ids that
      	can be passed to `DRM_IOCTL_MODE_GETPROPBLOB`
  - remember that these are property infos; the current value is returned by
    `DRM_IOCTL_MODE_OBJ_GETPROPERTIES`
- for example, a `DRM_MODE_OBJECT_PLANE` object has a property `type`
  indicating whether the plane is primary, cursor, or overlay
  - we list all properties first to get prop ids and prop values first
  - for each prop id, we get the info.  We are interested in the prop id whose
    name is `type`
  - the info should also contains a list of (enum name, enum val) enums
  - by comparing the prop value with the enum vals, we can find out the
    current enum name, which should be one of `Primary`, `Overlay`, or
    `Cursor`

## DRM modifiers

- To kernel,
  - `DRM_FORMAT_MOD_INVALID` (0x00ffffffffffffff) is truely invalid
  - `DRM_FORMAT_MOD_LINEAR` (0x0) is truely linear
  - in addfb2, when `DRM_MODE_FB_MODIFIERS` is not set, modifiers are not
    (explicitly) given.  modifiers array are required to be zeroed, but it is
    not the same as `DRM_FORMAT_MOD_LINEAR`
    - `intel_framebuffer_init` changes modifiers[0] when not explicitly given
      before calling `drm_helper_mode_fill_fb_struct` to set `fb->modifier`
  - a plane's `IN_FORMATS` never contains `DRM_FORMAT_MOD_INVALID`
- To EGL,
  - in eglCreateImage, when modifier is not explicitly given, it is considered
    to be `DRM_FORMAT_MOD_INVALID` and means impl-defined (similiar to addfb2
    without `DRM_MODE_FB_MODIFIERS`)
  - however, when `DRM_FORMAT_MOD_INVALID` is explicitly given, it becomes
    invalid
- xorg-modesetting allocates front buffers that are rendered to by glamor and
  scanned out by display engine
  - traditionally, set `GBM_BO_USE_RENDERING | GBM_BO_USE_SCANOUT` and be done
  - when supported, use modifiers instead
  - during crtc init, xorg-modesetting finds the best plane for the crtc
    - `drmmode_crtc_create_planes` also decodes the `IN_FORMATS` prop of the
      plane to get the list of supported formats and modifiers
    - virtio-gpu supports two planes
      - primary supports XR24
      - cursor supports AR24
    - i915 supports 9 planes
      - three primary supports CB RG16 XR24 XB24 XR30 XB30 XB4H (LINEAR or x-tiled)
      - three overlay supports XR24 XB24 XR30 XB30 XR4H XB4H YUYV YVYU UYVY VYUY (LINEAR or x-tiled)
      - three cursor supports AR24 (LINEAR)
  - bo allocation will pass in the modifiers supported by the crtc/plane
- glamor needs to allocate a pixmap or dmabuf-to-pixmap or pixmap-to-dmabuf
  - for alloc, `glTexImage2D`
  - for import, `gbm_bo_import`, `eglCreateImageKHR`, and
    `glEGLImageTargetTexture2DOES`
  - for export, `gbm_bo_create*`, import as pixmap, CopyArea, and replace the
    texture-based pixmap by the bo-based pixmap
  - EGL dmabuf extensions allow glamor to know which modifiers are supported
- gbm supports usage or modifiers, but not both
  - for usage, the dri backend only cares about `GBM_BO_USE_SCANOUT`,
    `GBM_BO_USE_CURSOR`, and `GBM_BO_USE_LINEAR`
  - i965's `intel_create_image_common` uses usage to pick between linear and
    x-tiled; for modifiers, it becomes explicit and can fail (when the
    modifier is not supported)
  - gallium drivers must implement `resource_create_with_modifiers` to support
    modifiers
  - gallium's `dri2_create_image_common` always sets `PIPE_BIND_RENDER_TARGET`
    and `PIPE_BIND_SAMPLER_VIEW`; it uses usage to set scanout/linear/cursor
    binds, which are not set with modifiers
- EGL supports modifiers
  - when importing a dma-buf, no modifier means implementation-defined
    modifier; `DRM_FORMAT_MOD_INVALID` is not a valid explicit modifier

## DRM formats

- `DRM_IOCTL_MODE_ADDFB2` takes a format instead of bpp/depth.  It also
  supports modifiers and planar formats.
  - internally, `drm_mode_legacy_fb_format` translates bpp/depth to format and
    `DRM_IOCTL_MODE_ADDFB` can take the same path as addfb2 does
- modifiers must be explicitly enabled by setting `DRM_MODE_FB_MODIFIERS` flag
