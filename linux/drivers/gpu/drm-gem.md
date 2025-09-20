DRM GEM
=======

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

## `drm_gem_shmem`

- public interface
  - macros
    - `to_drm_gem_shmem_obj` casts `drm_gem_object` to `drm_gem_shmem_object`
    - `DRM_GEM_SHMEM_DRIVER_OPS` is for `drm_driver` init
      - `drm_gem_shmem_dumb_create` for `drm_driver::dumb_create`, for
        `MODE_CREATE_DUMB` ioctl
      - `drm_gem_shmem_prime_import_no_map` for
        `drm_driver::gem_prime_import`, for `PRIME_FD_TO_HANDLE` ioctl
  - structs
    - `drm_gem_shmem_object` is a subclass of `drm_gem_object`
  - variables
    - `drm_gem_shmem_vm_ops` is for `drm_gem_object_funcs` init
      - it is set to `vma->vm_ops` upon mmap
  - functions
    - `drm_gem_shmem_init` inits an allocated gem obj
      - rust calls this
    - `drm_gem_shmem_create` allocs and inits gem obj
      - drivers call this on alloc
    - `drm_gem_shmem_create_with_mnt` allocs and inits gem obj
      - one driver calls this on alloc
    - `drm_gem_shmem_release` releases underlying resources
      - for imported gem obj, `drm_prime_gem_destroy` unmaps and detaches the
        dma-buf
      - for allocated gem obj, `shmem->sgt` and `shmem->pages` are teared down
      - `drm_gem_object_release` releases the base `drm_gem_object`
    - `drm_gem_shmem_free` releases and frees the gem obj
      - drivers call this from their `drm_gem_object_funcs::free`, which is
        `drm_gem_shmem_object_free` by default
    - `drm_gem_shmem_put_pages_locked` decrements `shmem->pages_use_count` and
      tears down `shmem->pages`
      - there is internal `drm_gem_shmem_get_pages_locked` that inits
        `shmem->pages` and increments `shmem->pages_use_count`
    - `drm_gem_shmem_pin` gets pages and increments `shmem->pages_pin_count`
      - pinned pages cannot be purged
    - `drm_gem_shmem_unpin` decrements `shmem->pages_pin_count` and puts pages
    - `drm_gem_shmem_vmap_locked`
      - for imported gem obj, `dma_buf_vmap` maps the dma-buf for kernel cpu
        access
      - for allocated gem obj, `drm_gem_shmem_pin_locked` pins pages and
        `vmap` maps the pages to `shmem->vaddr`
    - `drm_gem_shmem_vunmap_locked`
      - for imported gem obj, `dma_buf_vunmap` unmaps the dma-buf
      - for allocated gem obj, `vunmap` unmaps the pages and
        `drm_gem_shmem_unpin_locked` unpins pages
    - `drm_gem_shmem_mmap`
      - for imported gem obj, `dma_buf_mmap` mmaps the dma-buf
      - for allocated gem obj, `drm_gem_shmem_get_pages_locked` gets pages
    - `drm_gem_shmem_madvise_locked` sets `shmem->madv`
    - `drm_gem_shmem_is_purgeable` returns if the obj can be purged
    - `drm_gem_shmem_purge_locked` drops `shmem->sgt` and `shmem->pages`
      - it is for shrinker
    - `drm_gem_shmem_get_sg_table` allocs sgt for already pinned `shmem->pages`
      - the only external use should be for dma-buf, where `dma_buf_attach` is
        called before `dma_buf_map_attachment` to pin the pages
    - `drm_gem_shmem_get_pages_sgt` inits and returns `shmem->sgt`
      - for imported gem obj, `shmem->sgt` is already initialized from the
        dma-buf on import
      - for allocated gem obj,
        - `drm_gem_shmem_get_pages_locked` inits `shmem->pages`
        - `drm_gem_shmem_get_sg_table` allocs sgt and saves to `shmem->sgt`
    - `drm_gem_shmem_print_info`
    - `drm_gem_shmem_object_free` is for `drm_gem_object_funcs::free`
      - `drm_gem_object_put -> drm_gem_object_free` calls this
      - this calls `drm_gem_shmem_free`
    - `drm_gem_shmem_object_print_info` is for `drm_gem_object_funcs::print_info`
      - `drm_framebuffer_print_info -> drm_gem_print_info` calls this
      - this calls `drm_gem_shmem_print_info`
    - `drm_gem_shmem_object_pin` is for `drm_gem_object_funcs::pin`
      - `dma_buf_attach -> drm_gem_map_attach` calls this
      - this calls `drm_gem_shmem_pin_locked`
    - `drm_gem_shmem_object_unpin` is for `drm_gem_object_funcs::unpin`
      - `dma_buf_detach -> drm_gem_map_detach` calls this
      - this calls `drm_gem_shmem_unpin_locked`
    - `drm_gem_shmem_object_get_sg_table` is for `drm_gem_object_funcs::get_sg_table`
      - `dma_buf_map_attachment -> drm_gem_map_dma_buf` calls this
      - this calls `drm_gem_shmem_get_sg_table`
    - `drm_gem_shmem_object_vmap` is for `drm_gem_object_funcs::vmap`
      - `drm_gem_vmap` calls this to map for kernel cpu access
      - this calls `drm_gem_shmem_vmap_locked`
    - `drm_gem_shmem_object_vunmap` is for `drm_gem_object_funcs::vunmap`
      - `drm_gem_vunmap` calls this to unmap for kernel cpu access
      - this calls `drm_gem_shmem_vunmap_locked`
    - `drm_gem_shmem_object_mmap` is for `drm_gem_object_funcs::mmap`
      - `mmap -> drm_gem_mmap -> drm_gem_mmap_obj` calls this
      - `dma_buf_mmap -> drm_gem_dmabuf_mmap -> drm_gem_prime_mmap` calls this
      - this calls `drm_gem_shmem_mmap`
    - `drm_gem_shmem_prime_import_sg_table` is for
      `drm_driver::gem_prime_import_sg_table`
      - `drm_prime_fd_to_handle_ioctl -> drm_gem_prime_fd_to_handle -> drm_gem_prime_import`
        calls this
    - `drm_gem_shmem_dumb_create` is for `drm_driver::dumb_create`
      - `drm_mode_create_dumb_ioctl -> drm_mode_create_dumb` calls this
      - `drm_client_buffer_create -> drm_mode_create_dumb` calls this
    - `drm_gem_shmem_prime_import_no_map` is for
      `drm_driver::gem_prime_import`
      - `drm_prime_fd_to_handle_ioctl -> drm_gem_prime_fd_to_handle` calls this
      - "no map" means not calling `dma_buf_map_attachment` to get sgt for hw
        access, which is not typically used
