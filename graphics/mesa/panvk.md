Mesa PanVK
==========

## Meson

- `panvk_entrypoints` generates
  - `panvk_instance_entrypoints`
  - `panvk_physical_device_entrypoints`
  - `panvk_device_entrypoints`
  - `panvk_v6_device_entrypoints`
  - `panvk_v7_device_entrypoints`
  - `panvk_v9_device_entrypoints`
  - `panvk_v10_device_entrypoints`
  - the idea is that, for device entrypoints, it dispatches to `vk_vX_Foo` if
    defined and falls back to `vk_Foo` if not
- `panvk_per_arch_libs` are per-arch static libraries
  - the source files are
    - `common_per_arch_files` (`panvk_vX_*.c`)
    - `bifrost_files` (`bifrost/*.c`) if v7 or older
    - `valhall_files` (`valhall/*.c`) if v9 or newer
    - `jm_files` (`jm/*.c`) if v9 or older
    - `csf_files` (`csf/*.c`) if v10+
  - `PAN_ARCH` is defined per-arch
- `libvulkan_panfrost` is the final shared library
  - `libpanvk_files` consists of non-per-arch `panvk_*.c`
- depenencies
  - `idep_pan_packers`, packet packers generated by genxml
  - `libpanfrost_shared`, helpers shared between panvk, panfrost, and lima
  - `libpanfrost_midgard`, midgard (v5 or older) compiler
  - `libpanfrost_bifrost`, bifrost (v6 or newer) compiler
  - `libpanfrost_decode`, packet decoder generated by genxml
  - `libpanfrost_lib`, helpers shared between panvk and panfrost
  - `libpanfrost_util`, helpers shared between bifrost and midgard compilers

## Environment Variables

- `PAN_I_WANT_A_BROKEN_VULKAN_DRIVER` is required to use panvk
- `PANVK_DEBUG` is parsed by `panvk_debug_options`
  - `startup` prints init info
  - `nir` dumps lowered nir before passing it to the backend compiler
  - `sync` waits for all submits
  - `trace` implies `sync` and dumps all submits
  - `afbc` allows AFBC
  - `linear` forces linear tiling
  - `dump` dumps vk memory contents on submits
  - `no_known_warn` is JM (job manager) only
- `PANDECODE_DUMP_FILE` defaults to `pandecode.dump`
- `BIFROST_MESA_DEBUG` is for the bifrost compiler
- `MIDGARD_MESA_DEBUG` is for the midgard compiler
- drm-shim
  - `VK_DRIVER_FILES=panfrost_icd.x86_64.json`
  - `LD_PRELOAD=libpanfrost_noop_drm_shim.so`
  - `PAN_I_WANT_A_BROKEN_VULKAN_DRIVER=1`
  - `PAN_GPU_ID=0xABCD`
  - no panthor

## `pan_kmod_ops`

- `panthor_kmod_dev_create`
  - each `panvk_physical_device` and each `panvk_device` own a `pan_kmod_dev`
  - `DRM_IOCTL_PANTHOR_DEV_QUERY`
  - `drm_panthor_csif_info` on v10
    - `csg_slot_count = 8` means the fw supports 8 CSGs
      - if userspace creates more CSGs than the fw can support, the kernel
        driver will rotate between them
      - each `VkQueue` corresponds to a CSG
    - `cs_slot_count = 8` means there are 8 CSs per CSG
      - panvk creates 3 CSs per CSG, known as subqueues
      - each CS has its own `drm_gpu_scheduler`
    - `cs_reg_count = 96` means there are 96 cs regs per cs
      - panvk uses r66..r83 for scratch, r84..r89 for subqueue seqnos, and
        r90..r91 for subqueue ctx va
    - `scoreboard_slot_count = 8` means there are 8 scoreboard slots per cs
      - some cs cmds are async and increments a scoreboard slot
    - `unpreserved_cs_reg_count = 4` means the highest 4 cs regs are used by
      kernel
- `panthor_kmod_dev_destroy`
- `panthor_dev_query_props`
- `panthor_kmod_dev_query_user_va_range`
  - `panvk_device` uses this to decide the user va region
- `panthor_kmod_bo_alloc`
  - each `panvk_queue` has a bo as the ring buffer
  - `panvk_priv_bo_create` calls this for panvk-internal bos
  - each `panvk_device_memory` has a bo, either allocated or imported
  - `DRM_IOCTL_PANTHOR_BO_CREATE`
- `panthor_kmod_bo_free`
- `panthor_kmod_bo_import`
- `panthor_kmod_bo_export`
- `panthor_kmod_bo_get_mmap_offset`
  - `DRM_IOCTL_PANTHOR_BO_MMAP_OFFSET`
- `panthor_kmod_bo_wait`
  - panvk does not need this
- `panthor_kmod_vm_create`
  - each `panvk_device` has a vm
  - `DRM_IOCTL_PANTHOR_VM_CREATE`
- `panthor_kmod_vm_destroy`
  - `DRM_IOCTL_PANTHOR_VM_DESTROY`
- `panthor_kmod_vm_bind`
  - all bos must be bound to a device vm before they can be accessed by hw
  - `DRM_IOCTL_PANTHOR_VM_BIND`
- `panthor_kmod_vm_query_state`
  - panvk does not use this yet; query for faulty vm
  - `DRM_IOCTL_PANTHOR_VM_GET_STATE`
- `panthor_kmod_query_timestamp`
  - `DRM_IOCTL_PANTHOR_DEV_QUERY`
- some ioctls are not abstracted
- `panvk_per_arch(queue_init)`
  - `DRM_IOCTL_PANTHOR_TILER_HEAP_CREATE`
  - `DRM_IOCTL_PANTHOR_GROUP_CREATE`
    - `drm_panthor_group_create`
      - it can control which cores a draw or a dispatch runs on
      - `priority` decides the priority
      - `queues` uses 3 queues
        - `PANVK_SUBQUEUE_VERTEX_TILER`
        - `PANVK_SUBQUEUE_FRAGMENT`
        - `PANVK_SUBQUEUE_COMPUTE`
- `panvk_per_arch(queue_finish)`
  - `DRM_IOCTL_PANTHOR_GROUP_DESTROY`
  - `DRM_IOCTL_PANTHOR_TILER_HEAP_DESTROY`
- `panvk_queue_submit`
  - `DRM_IOCTL_PANTHOR_GROUP_SUBMIT`
- query device lost
  - `DRM_IOCTL_PANTHOR_GROUP_GET_STATE`

## Instance

- `panvk_CreateInstance`
  - the instance dispatch table combines
    - `panvk_instance_entrypoints`
    - `wsi_instance_entrypoints`
    - `vk_common_instance_entrypoints`
- `vk_common_EnumeratePhysicalDevices` calls
  `panvk_physical_device_try_create`
  - `enumerate_drm_physical_devices_locked` enumerates all drm devices
  - `VK_ERROR_INCOMPATIBLE_DRIVER` causes a drm device to be skipped and is
    not considered an error

## Physical Device

- `panvk_physical_device_init` inits a physical device from a drm device
  - `enumerate_drm_physical_devices_locked` silently ignores
    `VK_ERROR_INCOMPATIBLE_DRIVER`

## Formats and Modifiers

- format users
  - per-draw descriptors
    - `<struct name="Attribute" size="8" align="32">`
      - an `Attribute` corresponds to a vertex attrib
      - they are referred to by a `Resource` table (driver-internal descriptor
        set)
    - `<struct name="Texture" size="8" align="32">`
      - a `Texture` corresponds to a sampled image, storage image, or tbo/ibo
      - they are referred to by a `Resource` table
      - they are also referred to by `Shader Environment` of `Draw`
        - a `Draw` is a dcd pipeline for render pass load/store
        - because fb preload is sampling, it also implies that each
          color/depth att gets a `Texture`
    - `<struct name="Plane" size="8" align="32">`
      - a `Plane` corresponds to a "slice" of a `Texture`
        - there are `levels * layers * samples * planes` slices
      - they are referred to by a `Texture`
  - per-draw blend
    - `<struct name="Blend" size="4" align="16">`
      - a `Blend` corresponds to a color buffer
      - they are referred to by cs `d50` reg
      - they are also referred to by `Draw`, a dcd pipeline for render pass
        load/store
    - `<struct name="Internal Blend" align="8">`
      - an `Internal Blend` corresponds to a blend mode (fixed-function or
        shader)
      - it is embedded in a `Blend`
      - it is also hardcoded for each generated blend shader
        - blender is the hw unit to write back from tilebuffer to memory
        - blender only supports simple blend ops with 6 common formats
        - for unsupported blend ops or formats, fs calls a blend shader to
          perform the blending and to request blender to write back the
          blended value as an opaque value
  - per-render-pass framebuffer
    - `<aggregate name="Framebuffer" align="64">`
      - a `Framebuffer` corresponds to a render pass and consists of
        - `Framebuffer Parameters`
        - `Framebuffer Padding`
        - zero or one `ZS CRC Extension`
        - zero or more `Render Target`
      - it is referred to by cs `d40` reg
    - `<struct name="ZS CRC Extension" align="64" size="16">`
      - a `ZS CRC Extension` corresponds to a depth/stencil and/or crc buffer
      - it is referred to by `Framebuffer`
    - `<struct name="Render Target" align="64">`
      - a `Render Target` corresponds to a color buffer
      - it is referred to by `Framebuffer`
- different format users use different format enums
  - 22-bit `Pixel Format`
    - this is used by
      - `<struct name="Attribute" size="8" align="32">`
      - `<struct name="Texture" size="8" align="32">`
      - `<struct name="Internal Blend" align="8">`
      - different descriptors support different formats
    - bit 0..11 is `RGB Component Order`
    - bit 12..19 is `Format`
      - these can be roughly categorized into
        - color (renderable) formats
        - depth/stencil formats
        - compressed formats
        - planar formats
        - other formats (scaled, weird size, etc.)
      - all except other formats are usually texturable
      - color and other formats can be used as vertex formats
    - bit 20 is srgb
    - bit 21 must be 0 (it indicates big endian before v7)
  - `Clump Format` and `Clump Ordering`
    - only for non-AFxC `<struct name="Plane" size="8" align="32">`
    - what are these for?
  - `Register File Format`
    - only for `<struct name="Internal Blend" align="8">`
      - this tells the blender how to interpret the fs output value
    - F16, F32, I32, U32, I16, U16
  - `Color Buffer Internal Format`
    - only for `<struct name="Render Target" align="64">`
      - this is the tilebuffer color format
    - mostly RAW8, RAW16, RAW32, RAW64, RAW128 (block size in bits)
    - but also some 6 color formats supported by fixed-function blending
  - `Color Format`
    - only for `<struct name="Render Target" align="64">`
      - this is the writeback color format
    - mostly RAW8, RAW16, RAW24, RAW32, RAW48, RAW64, RAW96, RAW128, and all
      the way to RAW2048 (block size in bits)
    - but also some common color formats supported by fixed-function blending
  - `Z Internal Format`
    - only for `<struct name="Framebuffer Parameters" align="64">`
      - this is the tilebuffer depth format
      - the stencil format is always S8
    - D16, D24, and D32
  - `ZS Format` and `S Format`
    - only for `<struct name="ZS CRC Extension" align="64" size="16">`
      - they are the writeback depth/stencil formats
    - for depth: D16, D24X8, D24S8, and D32
    - for stencil: only S8 should be used; X24S8 is a waste
- Swizzles
  - 22-bit `Pixel Format` has `RGB Component Order` for limited swizzle
    - this is for, for example, `VK_FORMAT_R8G8B8A8_UNORM` and
      `VK_FORMAT_B8G8R8A8_UNORM` which differ only in component order
  - some structs also have a 12-bit `Swizzle` for arbitrary swizzle
    - `<struct name="Texture" size="8" align="32">`
    - `<struct name="Render Target" align="64">`
    - this is for `VkComponentMapping`
- Modifiers
  - `Plane Type`
    - this is only for `<struct name="Plane" size="8" align="32">`
    - generic, astc, afbc, afrc, chroma 2p
  - `Block Format`
    - this is for
      - `<struct name="Render Target" align="64">`
      - `<struct name="ZS CRC Extension" align="64" size="16">`
    - linear, tiled, afbc, afbc tiled
    - what is no write for?
- Clear Colors
  - `<struct name="Framebuffer Parameters" align="64">`
    - `Z Clear` is f32
    - `S Clear` is u8
  - `<struct name="Render Target" align="64">`
    - `Clear` is format-dependent
    - the hw memsets the tilebuffer to the opaque value
    - the driver must pack the clear value according to the rt format
- Blend Constants
  - `<struct name="Blend" size="4" align="16">`
    - `Constant` is a single 16-bit unorm
    - if the user-specified blend constant does not have the same values for
      all channels, it must fall back to the blend shader
- Border Colors
  - `<struct name="Sampler" size="8" align="32">`
    - `Border Color` is mempcy from user-specified `VkClearColorValue`
    - the user specifies `VkClearColorValue` as the border color
      - the format is typically unknown
      - the type (int or float) is known
      - when sampling from the border, the specified int or float value is
        returned
- YUV
  - `<struct name="Render Target YUV Overlay" size="16">`
    - this is for YUV rendering and can be ignored for now?
  - 22-bit `Pixel Format`
    - for RGB, this consists of
      - bit 0..11 is `RGB Component Order`
      - bit 12..19 is `Format`
      - bit 20 is srgb
      - bit 21 must be 0 (it indicates big endian before v7)
    - for YUV, this consists of
      - bit 0..2 is `YUV Swizzle`
      - bit 3 is swap
      - bit 4..8 must be 0
      - bit 9..11 is `YUV Cr Siting`
      - bit 12..19 is `Format`
      - bit 20..21 must be 0
  - `Clump Format` and `Clump Ordering`

## Device

- `panvk_CreateDevice` calls `panvk_per_arch(create_device)`
- `panvk_per_arch(create_device)`
  - the device dispatch table combines
    - `vk_cmd_enqueue_unless_primary_device_entrypoints`
      - this is for emulated secondary cmdbuf
      - a second dispatch table is intialized without this
      - when a `vkCmd*` is called on a primary cmdbuf, `vk_cmd_enqueue` uses
        the second dispatch table to dispatch immediately
      - when a `vkCmd*` is called on a secondary cmdbuf, `vk_cmd_enqueue`
        saves the call in a buffer
        - `vk_common_CmdExecuteCommands` replays the saved calls
    - `panvk_per_arch(device_entrypoints)`
    - `panvk_device_entrypoints`
    - `wsi_device_entrypoints`
    - `vk_common_device_entrypoints`
  - the render node fd is dupped to create a second `pan_kmod_dev`
    - this is wrong...
  - if decode is needed, `pandecode_create_context` creates a decode ctx
  - `util_vma_heap_init` inits `device->as.heap`
  - `pan_kmod_vm_create` creates `device->kmd.vm`
    - without `PAN_KMOD_VM_FLAG_AUTO_VA` such that the vm is managed by
      `device->as.heap`
  - `panvk_device_init_mempools` inits mempools
    - `dev->mempools.rw` is cached, for most stuff
    - `dev->mempools.rw_nc` is uncached, for event seqnos
    - `dev->mempools.exec` is executable, for shader binaries
  - `panvk_meta_init` inits `device->meta`
  - `panvk_per_arch(queue_init)` inits `device->queues`

## Queue

- `panvk_per_arch(queue_init)`
  - `drmSyncobjCreate` creates a syncobj
    - `queue->syncobj_handle` is used for semaphore signals
  - `init_tiler`
    - `tiler_heap->context` is from `DRM_IOCTL_PANTHOR_TILER_HEAP_CREATE`
    - `tiler_heap->desc` is a bo of size 4KB plus 64KB
      - the first 4KB is for `MALI_TILER_HEAP` descriptor
      - the rest 64KB is for geometry buffer
        - used by `MALI_PRIMITIVE_FLAGS` fifo
  - `create_group`
    - `queue->group_handle` is from `DRM_IOCTL_PANTHOR_GROUP_CREATE`
    - there are 3 sub-queues
      - `PANVK_SUBQUEUE_VERTEX_TILER`
      - `PANVK_SUBQUEUE_FRAGMENT`
      - `PANVK_SUBQUEUE_COMPUTE`
  - `init_queue`
    - `queue->syncobjs` is an array of per-subqueue `panvk_cs_sync64`
      - this is to track the per-subqueue seqno
    - `init_render_desc_ringbuf` inits `queue->render_desc_ringbuf`
      - this is used only for `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`
      - `ringbuf->bo` is `RENDER_DESC_RINGBUF_SIZE` (512KB)
      - `ringbuf->bo` is vm-bound twice consecutively
        - this simplifies wrap-around handling
      - `ringbuf->syncobj` is a `panvk_cs_sync32`
    - `init_subqueue` inits each of `queue->subqueues`
      - `subq->context` is a bo for `panvk_cs_subqueue_context`
        - it is only accessed by hw after initialization
        - `syncobjs` points to `queue->syncobjs` and `syncobjs[subq]` is
          initialized to 1
        - `iter_sb` is initialized to 0
        - `render` is not used by `PANVK_SUBQUEUE_COMPUTE`
          - `tiler_heap` points to `queue->tiler_heap.desc`
          - `geom_buf` points to `queue->tiler_heap.desc + 4KB`
          - `desc_ringbuf` points to `queue->render_desc_ringbuf`
      - first submit to init the subqueue
        - this abuses the geom buf as cs root chunk
        - `MOV64 cs_subqueue_ctx_reg(), subq->context` inits the subq ctx reg
        - `SET_SB_ENTRY` such that ep tasks (frag & comp) use `SB_ITER(0)`
          (slot 3) and other tasks (tiler) uses `SB_ID(LS)` (slot 0)
          - this matches `iter_sb` in subq ctx
        - if not `PANVK_SUBQUEUE_COMPUTE`, inits the tiler heap as well
          - `MOV64 scratch0, queue->tiler_heap.context.dev_addr`
          - `HEAP_SET scratch0`
- `panvk_queue_submit`
  - if there are semaphore waits
    - sets up a `drm_panthor_sync_op` with `DRM_PANTHOR_SYNC_OP_WAIT` for each
      wait
    - sets up an empty `drm_panthor_queue_submit` for each used subqueue to
      wait on the `drm_panthor_sync_op` array above
  - for each cmdbufs
    - sets up a `drm_panthor_queue_submit` for each used subqueue
  - if there are semaphore signals
    - for each used subqueue, sets up a `drm_panthor_sync_op` with
      `DRM_PANTHOR_SYNC_OP_SIGNAL` and sets up an empty
      `drm_panthor_queue_submit`
    - they all signal `queue->syncobj_handle`
    - the fence is then copied to user semaphores using `drmSyncobjTransfer`
    - `queue->syncobj_handle` is reset
- flush id
  - `CSF_GPU_LATEST_FLUSH_ID` reg
    - if unmapped, kernel `panthor_mmio_vm_fault` maps the reg or a dummy page
      which contains flush id 1
    - on device suspend/resume, the mapping is unmapped
    - the idea is to return the real flush id when the device is powered, and
      to return 1 when the device is not powered
  - `panvk_per_arch(EndCommandBuffer)`
    - `panvk_per_arch(CmdPipelineBarrier2)` calls `cs_flush_caches` with flush
      id 0, to flush unconditionally
    - `finish_cs` calls `cs_flush_caches` with flush id 0, to flush
      unconditionally
    - `cmdbuf->flush_id` is read from `CSF_GPU_LATEST_FLUSH_ID` reg
  - `panvk_queue_submit` submits each job with `cmdbuf->flush_id`
    - kmd flushes caches with the specified flush id before calling into the
      job
  - from `drm_panthor_queue_submit` doc,
    - kmd emits a flush to avoid reading staled data
    - the flush can be eliminated if the flush id is properly setup
  - still no idea what it does

## BO and Address Space

- `pan_kmod_bo_alloc` makes the `DRM_IOCTL_PANTHOR_BO_CREATE` ioctl
  - there are only 3 callers
  - `init_queue` allocs a ringbuf for
    `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`
  - `panvk_AllocateMemory` allocs a bo for user device memory
  - `panvk_priv_bo_create` allocs a bo for driver use, which in turn have
    these callers
    - `panvk_pool_alloc_backing` allocs a bo for a mempool
    - `panvk_per_arch(CreateDescriptorPool)` allocs a bo for descs
    - `panvk_per_arch(create_device)` allocs a bo for sample pos
- `pan_kmod_vm_bind` makes the `DRM_IOCTL_PANTHOR_VM_BIND`
  - this has the same callers as `pan_kmod_bo_alloc` does at the moment, plus
    unmaps on destroy
  - because `PAN_KMOD_VM_FLAG_AUTO_VA` is not set when creating
    `dev->kmod.vm`, all vmas are allocated from
    `util_vma_heap_alloc(&dev->as.heap)`
- `panvk_per_arch(create_device)` inits the vm
  - `user_va_start` is 32MB
  - `user_va_end` is 4GB
  - `device->as.heap` manages `[user_va_start, user_va_end)`
    - by default, `util_vma_heap` allocs from the top
    - with 4GB AS, bos start from `0xffffffff` and grow down
  - `device->kmod.vm` is created with the same range
    - `vm_flags` does not include `PAN_KMOD_VM_FLAG_AUTO_VA`
      - if it did, kmod would init a `util_vma_heap` the same way
    - hw reports `mmu_features`
      - typically, there are 40 pa bits and 48 va bits
      - the 48 va space is shared by usersapce (bottom) and kernel space (top)
      - because panvk asks for 4GB, the first 4GB is reserved for userspace
        and the kernel uses the rest
      - `panthor_vm_create_check_args` wants the start addr to be
        power-of-two and ends up with `1ull << 47`
      - `drm_mm` with `DRM_MM_INSERT_BEST` allocs from the bottom
      - bos thus start from `0x800000000000` and grows up

## BO Pool

- types
  - `pan_pool` is the base class of `panvk_pool`, which is not too useful
  - `panvk_bo_pool` is an optional bo cache
  - `panvk_pool` is the bo suballocator
    - if `pool->props.owns_bos`, the pool tracks all bos and is responsible
      for freeing them
    - if `pool->bo_pool`, the pool moves bos to the bo pool (for reuse) rather
      than freeing them
- `panvk_pool_alloc_backing` makes a bo allocation
  - if there is `pool->bo_pool`, it tries to take a bo from the cache
  - otherwise, it allocs a new bo
  - if `pool->props.owns_bos`, the bo is tracked by the pool
  - `pool->transient_bo` is set to the new bo
    - this is the current bo that the pool suballocates from
- `panvk_pool_alloc_mem` suballocs a bo
  - if `pool->transient_bo` does not have enough space, it calls
    `panvk_pool_alloc_backing` to alloc a new one
- `panvk_pool_reset` and `panvk_pool_cleanup`
  - if `pool->props.owns_bos`, bos are tracked and are freed now
    - if there is `pool->bo_pool`, bos are moved to it for reuse instead
  - else, only `pool->transient_bo` is freed

## Memory

- `panvk_AllocateMemory`
  - `mem->bo` is allocated by `pan_kmod_bo_alloc`
    - normally, the bo is exclusive to `device->kmod.vm`
    - if export, there is no exclusive vm
    - if import, `pan_kmod_bo_import` is used instead
  - `util_vma_heap_alloc` allocs the va
  - `pan_kmod_vm_bind` binds the bo to the va

## Buffer

- `panvk_GetBufferMemoryRequirements2`
  - alignment is 64
  - size is aligned to 64
- `panvk_BindBufferMemory2`
  - `dev_addr` is the base addr of the bo plus the bind offset
  - `bo` is the bo
  - `host_ptr` is the bo mapping, if the buffer is an index buffer
    - this is for before v10
- `panvk_per_arch(CreateBufferView)`
  - if the buffer is used as a tbo or ibo, it is treated as a 1d image
    - `mem` and `descs.tex` are for the 1d image descriptor
- `panvk_per_arch(UpdateDescriptorSets)`
  - `write_buffer_view_desc` is for tbo/ibo
    - it copies `view->descs.tex` to the descriptor
  - `write_buffer_desc` is for regular buffers
    - it inits `MALI_BUFFER` on the descriptor
  - `write_dynamic_buffer_desc` is for dynamic buffers
    - it remebers the buffer at `set->dyn_bufs`
    - on each draw/dispatch, a "driver descriptor set" will be allocated and
      it will contain the descriptors of dynamic buffers

## Image

- `panvk_CreateImage`
  - `image->pimage.layout` is partially initialized from `VkImageCreateInfo`
    - format, dim
    - width, height, depth
    - mip levels (called slices), array size
    - sample count
  - `panvk_image_pre_mod_select_meta_adjustments` adjusts image usage/flags
  - `panvk_image_select_mod` selects the modifier
  - `pan_image_layout_init` fully initializes the layout
    - it can derive the physical layout from the logical layout
    - it also supports explicit layout
    - `layout->data_size` is the total size of the image
    - `layout->array_stride` is the stride between two array layers
      - each array layer is a full mipmap
    - `slice->size` is the total size of a slice (miplevel)
    - `slice->surface_stride` is the stride between two surfaces
      - a slice has `depth * layout->nr_samples` surfaces
- `panvk_BindImageMemory2`
  - `bo` points to the memory bo
  - `image->pimage.data` is the bo addr and the bind offset
- `panvk_per_arch(CreateImageView)`
  - if the image is sampled/storage/input, it requires a descriptor
    - a `MALI_TEXTURE`, whose `surfaces` field points to an array of
      `MALI_PLANE`
    - `mem` is the gpu bo for `MALI_PLANE` array
    - `descs.tex` is `MALI_TEXTURE` to be copied into descriptor set
  - `GENX(panfrost_estimate_texture_payload_size)` estimates the size of the
    `MALI_PLANE` array
    - `sizeof(MALI_PLANE) * planes * levels * layers * samples`
  - `GENX(panfrost_new_texture)` inits the descriptor
    - `panfrost_emit_texture_payload` inits `MALI_PLANE` array
      - for each layer
      - for each sample
      - for each level
      - for each plane
      - init `MALI_PLANE`
    - `MALI_TEXTURE` is initialized as well
- `panvk_per_arch(UpdateDescriptorSets)`
  - `write_image_view_desc` copies `view->descs.tex` to the descriptor

## Modifier

- mali supports these modifiers
  - `DRM_FORMAT_MOD_LINEAR`
  - `DRM_FORMAT_MOD_ARM_16X16_BLOCK_U_INTERLEAVED`
  - `DRM_FORMAT_MOD_ARM_AFBC(mode)`
    - bit 0..3: block size
      - 16x16, 32x8, 64x4
      - there is also `32x8_64x4` when lumi and chroma planes have different
        block sizes
    - bit 4: YTR, uses lossless color transform
      - this is only supported by rgb and should be set for rgb for better
        compression
    - bit 5: SPLIT, splits the payload of each block
    - bit 6: SPARSE
    - bit 7: CBR, copy-block restrict
    - bit 8: TILED, groups blocks into 8x8 or 4x4 tiles
      - this requires v7+
    - bit 9: SC, uses solid color blocks
      - this requires v7+
    - bit 10: DB, indicates front-buffer rendering safe
    - bit 11: BCH, uses buffer content hint
    - bit 12: USM, uses uncompressed storage mode
  - `DRM_FORMAT_MOD_ARM_AFRC(mode)`
    - bit 0..3: coding unit for plane 0
      - compress every N bytes to a fixed compressed size
    - bit 4..7: coding unit for plane 1 and 2 
    - bit 8: SCANOUT, uses scanline layout rather than rotated layout
- each rt has a `MALI_RENDER_TARGET`
  - `pan_emit_rt` emits the descriptor
  - if afbc,
    - dw 1: block format, etc.
    - dw 2: comp mode, ytr, wide block, etc.
    - dw 8..9: va of afbc header
    - dw 10: row stride
    - dw 11: offset of afbc body relative to header
- zs has `MALI_ZS_CRC_EXTENSION`
  - `pan_emit_zs_crc_ext` emits the descriptor
  - if afbc,
    - dw 0: block format, etc.
    - dw 8..9: va of afbc header
    - dw 10: row stride
    - dw 11: offset of afbc body relative to header
- some images/buffers have `MALI_PLANE`s
  - `panfrost_emit_plane` emits the descriptor
  - if afbc,
    - dw 0: plane type, super block size, ytr, split, alpha hint, tiled, comp
      mode, header size
    - dw 2..3: va of afbc header
    - dw 4: row stride
- `pan_best_modifiers` is an ordered list of best modifiers
  - afbc, 16x16, tiled, sc, sparse, ytr
  - afbc, 16x16, tiled, sc, sparse
  - afbc, 16x16, sparse, ytr
  - afbc, 16x16, sparse
  - `DRM_FORMAT_MOD_ARM_16X16_BLOCK_U_INTERLEAVED`
  - `DRM_FORMAT_MOD_LINEAR`
  - afrc

## Sampler

- `panvk_per_arch(CreateSampler)`
  - it inits a `MALI_SAMPLER`
- `panvk_per_arch(UpdateDescriptorSets)`
  - `write_sampler_desc` copies `sampler->desc` to the descriptor

## Descriptor

- `panvk_per_arch(CreateDescriptorSetLayout)`
  - `vk_create_sorted_bindings` seems redundant
  - it parses `pCreateInfo->bindings` to `layout->bindings`
    - `immutable_samplers` is the hw descs copied from `panvk_sampler`
    - `desc_idx` is the desc index in the descriptor set
      - dynamic buffers use a different namespace (because they use "push
        set")
      - combined image sampler uses 2 descriptors
  - `layout->hash` is unused
    - it should use `layout->vk.blake3` instead?
- `panvk_per_arch(CreateDescriptorPool)`
  - `pool->sets` is the desc sets, all pre-allocated
  - `pool->free_sets` is a bitmask, indicating which sets are free
  - `pool->desc_bo` is the bo for hw descs
  - `pool->desc_heap` manages the vma for the bo
- `panvk_per_arch(AllocateDescriptorSets)`
  - it takes a free set from `pool->sets`
  - it reserves a region from `pool->desc_heap`
  - it refs the layout
  - it writes descs for immutable samplers, if any
- `panvk_per_arch(FreeDescriptorSets)`
  - it marks the set free in `pool->free_sets`
  - it returns the region to `pool->desc_heap`
  - it unrefs the layout
- `panvk_per_arch(ResetDescriptorPool)` frees all desc sets and marks them
  free
- `panvk_per_arch(UpdateDescriptorSets)`
  - `write_sampler_desc`
    - it writes `sampler->desc` for each sampler
  - `write_image_view_desc`
    - it writes `view->descs.tex` for each view
    - if combined sampler/image, it also writes the sampler desc
  - `write_buffer_view_desc`
    - it writes `view->descs.tex` for each view
  - `write_buffer_desc` is for untyped buffers
    - it writes a `MALI_BUFFER` for each buffer
  - `write_dynamic_buffer_desc` is for dynamic buffers
    - they use "push sets" and only the vas are saved to `set->dyn_bufs`

## Event

- a `panvk_event` has
  - `syncobjs`, a bo for `PANVK_SUBQUEUE_COUNT` `panvk_cs_sync32`
- `panvk_per_arch(SetEvent)` sets all seqnos to 1
- `panvk_per_arch(ResetEvent)` sets all seqnos to 0
- `panvk_per_arch(GetEventStatus)` checks if all seqnos are non-zero
- `panvk_per_arch(CmdSetEvent2)` emits `SYNC_SET32` to set all seqnos to 1
- `panvk_per_arch(CmdResetEvent2)` emits `SYNC_SET32` to set all seqnos to 0
- `panvk_per_arch(CmdWaitEvents2)` emits `SYNC_WAIT32` to watit util seqnos
- `panvk_per_arch(get_cs_deps)` gets the deps for synchronization
