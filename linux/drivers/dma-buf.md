dma-buf
=======

## History

- dma-buf in 2011 by Linaro for cross-device buffer sharing
  - d15bd7ee445d0702ad801fdaece348fdb79e6581
- drm prime in 2012 by RedHat to make use of dma-buf
  - 3248877ea1796915419fba7c89315fdbf00cb56a
  - for GPU offloading (rendering in one, presenting in another)
- v4l2 adoption of dma-buf in 2012 by Linaro
  - c538404869b69db543ce23cf041192c192a65330
  - for GPU texturing from camera output
  - there was no fencing; I guess userspace must wait for camera/gpu to idle
    before sampling/recycling
- dma-resv in 2013 by Canonical
  - 786d7257e537da0674c02e16e3b30a44665d1cee
  - derived from `ttm_bo_reserve`
  - only `ww_mutex` to allow bo locking with deadlock avoidance
- dma-fence in 2014 by Canonical
  - e941759c74a44d6ac2eed21bb0a38b21fe4559e2
  - dma-fence was also added to dma-resv
  - dma-resv was also added to dma-buf
  - cross-device implicit fencing was made possible
  - userspace can also poll (the implicit fences of) a dma-buf
- dma-buf cpu ioctl in 2016 by Intel
  - c11e391da2a8fe973c3c2398452000bed505851e
- sync-file in 2016 by Collabora
  - cross-device explicit fencing was made possible

## dma-fence

- `struct dma_fence` has
  - `lock` for a spinlock to protect `cb_list`
    - it is locked to protect `cb_list` in
      - `dma_fence_add_callback`
      - `dma_fence_remove_callback`
      - `dma_fence_signal` and `dma_fence_signal_timestamp`
    - it is locked in many more functions, but all of them accesses `cb_list`
      - as a result, it is locked in cbs, in several ops, and is often locked
        beyond `cb_list` access
  - `ops` for impl-defined ops
      - `get_driver_name`, unlocked
      - `get_timeline_name`, unlocked
      - `enable_signaling`, locked
      - `signaled`, unlocked
      - `wait`, unlocked, deprecated by `dma_fence_default_wait`
      - `release`, unlocked, default to `dma_fence_free`
      - `fence_value_str`, locked
      - `timeline_value_str`, locked
      - `set_deadline`, unlocked
  - a union of
    - `cb_list` for a list of callbacks to be invoked when the fence signals
      - valid from `dma_fence_init` until `dma_fence_signal`
    - `timestamp` to indicate when it was signaled
      - valid from `dma_fence_signal` until `dma_fence_release`
    - `rcu` for use with `kfree_rcu`
      - valid after `dma_fence_release`
      - dma-fence is designed to be RCU-protected
        - A writer can `dma_fence_put` without `synchronize_rcu`.  This is
          because when the last reference is dropped, the fence is freed with
          `kfree_rcu`
        - on the other hand, a reader might get a fence with `refcount==0`.
          It must call `dma_fence_get_rcu` to make sure the fence is usable.
  - `context` for the fence context
  - `seqno` for the seqno of the fence in the fence context
  - `flags` for its internal status
    - `DMA_FENCE_FLAG_SIGNALED_BIT` indicates fence is already signaled
    - `DMA_FENCE_FLAG_TIMESTAMP_BIT` indicates fence has a valid timestamp
      - this is needed because there is a small window where `cb_list` is
        saved away and `timestamp` becomes valid in the union
    - `DMA_FENCE_FLAG_ENABLE_SIGNAL_BIT` indicates signaling is enabled
      - dma-fence may not signal timely.  To get that behavior, which may
        require enabling IRQ, one of these should be called
        - `dma_fence_add_callback`
        - `dma_fence_wait*`
        - `dma_fence_enable_sw_signaling`
    - more impl-defined flags
  - `refcount` for refcount
  - `error` to indicate an error in the associated operation
    - it must be set before signaling
- `dma_fence_get_stub` returns a fence that is already signaled
- `dma_fence_context_alloc` allocs fence contexts (u64 ids)
- `dma_fence_wait` and `dma_fence_wait_timeout` wait for a fence
  - `dma_fence_enable_sw_signaling` enables signaling (irq if necessary)
  - `dma_fence_default_wait`
    - it early returns in several cases
      - the fence is already signaled
      - the wait is interruptible and there is a pending process signal
      - timeout is 0
    - otherwise, it adds a `dma_fence_default_wait_cb` to `cb_list` and enters
      a sleep loop until the fence is signaled or interrupted
      - `dma_fence_default_wait_cb` is responsible for waking up self
- `dma_fence_wait_any_timeout` waits for multiple fences
- when the associated operation completes, driver calls `dma_fence_signal*` to
  signal the fence
  - the lock is locked or already locked
  - it saves `cb_list` on stack
    - remember that `cb_list` and `timestamp` are in a union
  - it sets `timestamp` and `DMA_FENCE_FLAG_TIMESTAMP_BIT`
    - check `dma_fence_timestamp` out to see why the bit is needed
  - it loops through `cb_list`, reinits the cb nodes, and invokes cbs
- fence cross-driver contract
  - fences must complete in a reasonable time
    - if the associated operation runs forever, there must be a timeout
  - fence timeout is impl-defined
    - it could be a fixed timeout, or it could be a mix of forward progress
      and dynamic timeout
  - driver should annotate all code leading to `dma_fence_signal` with
    `dma_fence_begin_signalling` and `dma_fence_end_signalling`
    - this detects potential deadlock around `dma_fence_wait`
  - driver is allowed to `dma_fence_wait` while holding `dma_resv_lock`
    - this means all code leading to `dma_fence_signal` must not
      `dma_resv_lock`
  - driver is allowed to `dma_fence_wait` from shrinker
    - this means all code leading to `dma_fence_signal` must not
      allocate with `GFP_KERNEL`
  - driver is allowed to `dma_fence_wait` from their
    `mmu_interval_notifier_ops` (for userptr)
    - this means all code leading to `dma_fence_signal` must not
      allocate with `GFP_NOFS` or `GFP_NOIO`, but only `GFP_ATOMIC`

## dma-resv

- `struct dma_resv` has
  - `lock` protects `fences`
    - only for writers; readers are lockfree
  - `fences`
    - `rcu`
    - `num_fences` is array size
    - `max_fences` is array capacity
    - `table` is an array of `dma_fence` and usage
      - usage is stored in lower bits (`DMA_RESV_LIST_MASK`) of the pointer
        - `DMA_RESV_USAGE_KERNEL` the associated op is for kernel mm
        - `DMA_RESV_USAGE_WRITE` the associated op is userspace write
        - `DMA_RESV_USAGE_READ` the associated op is userspace read
        - `DMA_RESV_USAGE_BOOKKEEP` the associated op is not implicitly fenced
        - the enums are ordered such that the lower ones "implies" the higher
          ones
        - for a driver that does not support implicit fencing,
          `DMA_RESV_USAGE_BOOKKEEP` is the only usage
- `dma_resv_init` inits an dma-resv
  - `ww_mutex_init` inits `obj->lock`
  - `RCU_INIT_POINTER` inits `obj->fences`
- `dma_resv_lock` locks the dma-resv
  - `ww_mutex_lock` locks `obj->lock`
  - this is a `ww_mutex` because the gpu driver needs to lock dma-resv of all
    bos for a submit
  - only writers lcok dma-resv; readers are lockfree
- `dma_resv_reserve_fences` ensures storage
  - it early returns if there is enough storage
  - `dma_resv_list_alloc` allocs new storage
  - it copies existing fences, if any, from old storage to new storage
    - it calls `dma_fence_is_signaled`, which can signal the fence, and drops
      signaled fence
- `dma_resv_add_fence` adds a fence
  - the fence should not be `dma_fence_is_container`
  - it checks if it can replace an existing fence
    - if the new fence is later in the same fence context than an existing
      fence, and has a stronger usage, it can replace the existing fence
    - it also calls `dma_fence_is_signaled`
  - otherwise, it appends the fence
  - this function cannot fail because it is called from a point of no failure
    - `dma_resv_reserve_fences` must have been called to ensure storage
- `dma_resv_replace_fences` replaces all fences in the specified context

## dma-fence Containers

- `dma_fence_array`
  - it is a fence which consists of a fixed-size array of fences
    - the fence may be on a different timeline because the underlying fences
      may be from different timelines
  - this is used by `sync_file`
- `dma_fence_chain`
  - it is a fence that wraps another fence and supports chaining
  - this is used by timeline syncobj
    - the syncobj is itself a timeline (rather a point on another timeline)
    - the timeline consists of discrete fences, which are points on the
      timeline, and are managed with a dma-fence-chain
    - userspace can
      - userspace can advance the seqno of the timeline
        - appends a signaled fence with the desired seqno to the chain
      - userspace can query the seqnos of the points
        - get seqnos of the fences
      - userspace can wait until the timeline reaches a certain seqno
        - wait for fences before the seqno to signal

## sync-file

- a `struct file` backed by anonymous inode that wraps a dma-fence
- it enables userspace to operate on the dma-fence using `poll`, `ioctl`, as
  well as transferring the dma-fence to another process
- it is possible to create a sync-file by merging two sync-files.  The new
  sync-file wraps a dma-fence-array.

## sw-sync

- `CONFIG_SW_SYNC` creates `/sys/kernel/debug/sync`
  - `sw_sync` supports ioctls to create dma-fences
  - `info` dumps `sync_timeline_list_head` and `sync_file_list_head`
    - `sync_timeline_list_head` is only used by `sw_sync`
    - `sync_file_list_head` is unused
- `sw_sync_debugfs_fops`
  - `sw_sync_debugfs_open` associates a `sync_timeline` with the file
    - `context` is from `dma_fence_context_alloc`
    - `name` is from `get_task_comm`
    - `pt_list` is a list of all `sync_pt` sorted by seqnos
    - `pt_tree` is an rb tree to help find the right spot to insert a new
      `sync_pt` to `pt_list`
  - `sw_sync_ioctl`
    - `SW_SYNC_IOC_CREATE_FENCE` creates a `sync_pt` and a `sync_file`
      - `sync_pt_create` create a `sync_pt`, which is a subclass of
        `dma_fence` with a userspace-specified 32-bit seqno
        - the `sync_pt` is added to `pt_list`, which is sorted by seqnos
      - `sync_file_create` creates a `sync_file` to wrap `sync_pt`
    - `SW_SYNC_IOC_INC` increments `sync_timeline`
      - `sync_timeline_signal` increments the timeline value
        - all `sync_pt` with seqnos less than or equal to the timeline value
          are signaled and removed
  - `sw_sync_debugfs_release` signals all remaining `sync_pt` before freeing
    the timeline
- `timeline_fence_ops` is the `dma_fence_ops` for `sync_pt`
  - `timeline_fence_get_driver_name` returns `sw_sync`
  - `timeline_fence_get_timeline_name` returns the name of `sync_timeline`,
    which is the comm name
  - `timeline_fence_signaled` returns true if the timeline value is greater
    than or equal to the fence seqno
  - `timeline_fence_release` frees the `sync_pt`

## dma-buf

- Introduction
  - A device driver can export its DMA buffer as a dma-buf by calling
    `dma_buf_export`
  - The dma-buf can be passed to userspace as a fd by calling `dma_buf_fd`
  - userspace can share this fd with another process/device
  - the other end can call `dma_buf_get` to get the dma-buf back and call
    `dma_buf_attach` to add itself to the dma-buf's attachment list
  - the other end can get the `sg_table` for DMA access to the dma-buf by
    calling `dma_buf_map_attachment` and `dma_buf_unmap_attachment`
  - when done with the dma-buf, `dma_buf_detach` and `dma_buf_put` should be
    called
- `dma_buf_export` create a `struct dma_buf`
  - there is always a `struct dma_resv` for implicit fencing
  - this creates a `struct file` whose `private_data` points back to the
    dma-buf
  - there is `priv` which normally points back to the device driver buffer
    object
- userspace can
  - `lseek` on the fd to get the buffer size
  - `poll` the fd
    - readable when the exclusive fence is signaled
    - writable when all fences are signaled
    - this indicates the DMA tranfers are complete.  Cache flushing might be
      required before CPU access.
  - `mmap` the fd to map the DMA buffer; no synchronization
  - `DMA_BUF_SET_NAME` to assign a name for debugging
  - `DMA_BUF_IOCTL_SYNC` to sync for CPU access
    - `DMA_BUF_SYNC_START` start CPU access
      - usually involves waiting and cache flushing
    - `DMA_BUF_SYNC_READ` / `DMA_BUF_SYNC_WRITE` sync for cpu read / write
      - controls waiting on only exclusive fence or all fences
    - `DMA_BUF_SYNC_END` end CPU access
      - usually involves cache flushing
- `dma_buf_get` and `dma_buf_attach`
  - the other driver is given an fd
  - calls `dma_buf_get` to map it to the dma-buf
  - calls `dma_buf_attach` to notify the original driver the intention to use
  - creates a other-driver-specific buffer object that wraps the dma-buf
- `dma_buf_map_attachment`
  - the other driver calls `dma_buf_map_attachment` to get the `sg_table` before
    DMA
  - the original driver duplicates its `sg_table` for the dma-buf and calls
    `dma_map_sg` to get the dma addresses for the pages
  - this other driver wraps its buffer object around the `sg_table`
- `dma_buf_unmap_attachment` is the opposite of mapping
- `dma_buf_detach`
- dma-buf provides other functions for kernel space CPU fallback
  - `dma_buf_begin_cpu_access` / `dma_buf_end_cpu_access`
  - `dma_buf_kmap` / `dma_buf_kunmap`
  - `dma_buf_vmap` / `dma_buf_vunmap`
- Driver Device Access
  - `dma_buf_ops::attach` and `dma_buf_dynamic_attach`
    - when a device wants access to a dma-buf, `dma_buf_dynamic_attach` is
      called; the exporter should check whether the dma-buf is dma-able by the
      given device (as-is or after moving the dma-buf to a different heap)
  - `dma_buf_ops::map_dma_buf` and `dma_buf_map_attachment`
    - this maps a dma-buf into a device's address space and returns an
      `sg_table`; the exporter may sleep and potentially move the dma-buf (from
      vram to system ram for example)
- Driver CPU Access
  - `dma_buf_ops::vmap` and `dma_buf_vmap`
    - when a device driver wants internal CPU access to an entire dma-buf,
      `dma_buf_vmap` is called; the exporter returns a coherent(?) CPU mapping
  - alternatively, the `sg_table` can be `kmap`ed and
    `dma_buf_begin_cpu_access` can be used for coherency
- Userspace CPU Access
  - `dma_buf_ops::mmap` and `dma_buf_mmap`/`dma_buf_mmap_internal`
    - when a device driver provides its own mechansim for userspace mmap, the
      driver can call `dma_buf_mmap`
    - the dma-buf fd can also be directly mmaped and the dma-buf core calls
      `dma_buf_mmap_internal`
    - the expoter returns a CPU mapping that might NOT be coherent
  - `dma_buf_ops::begin_cpu_access` and `DMA_BUF_IOCTL_SYNC`
    - CPU access might be bracketed by begin/end cpu access to give the
      expoter a chance to flush/invalidate CPU caches.
- storage pinning
  - a device driver bo is normally backed by a shmem.  The backing pages are
    allocated lazily and can be swapped out anytime.  Only when the shmem is
    in use by the device, the driver pins the pages.
  - similar to mlock, unprivileged userspace should NOT be able to pin pages
    indefinitely
  - a dma-buf exporter should pin pages in `dma_buf_ops::map_dma_buf`; a
    dma-buf importer should map dma-buf only when the pages need to be pinned
  - for cpu access, when a dma-buf is mmaped, the exporter can mmap the shmem;
    there is no need to pin the pages.
- dynamic mapping
  - this adds `dma_buf_ops::pin` and decouples pinning from
    `dma_buf_ops::map_dma_buf`
  - when both the importer and the exporter support dynamic mapping, a mapping
    might move anytime
    - the importer will be notified by `dma_buf_attach_ops::move_notify`

## dma-buf heap

- system heap
  - `system_heap_create` calls `dma_heap_add` to create `system` heap
  - `system_heap_allocate` allocates a dma-buf
    - `alloc_largest_available` calls `alloc_pages` to alloc pages
    - `sg_alloc_table` and `sg_set_page` init `buffer->sg_table` from the
      pages
    - `dma_buf_export` exports the dma-buf with `system_heap_buf_ops`
    - I suppose it is not possible to swap out the pages
  - `system_heap_attach` duplicates `buffer->sg_table` and tracks the
    attachment in `buffer->attachments`
  - `system_heap_detach` undoes attach
  - `system_heap_map_dma_buf` calls `dma_map_sgtable` to map the duplicates sg
    table (to device addr space) and returns the table
  - `system_heap_unmap_dma_buf` calls `dma_unmap_sgtable`
  - `system_heap_dma_buf_begin_cpu_access` calls `dma_sync_sgtable_for_cpu`
    on all duplicated sg tables
    - in case the buffer is vmapped in kernel, `invalidate_kernel_vmap_range`
      is also called to invalidate the cpu cache
  - `system_heap_dma_buf_end_cpu_access` calls `dma_sync_sgtable_for_device`
    on all duplicated sg tables
    - in case the buffer is vmapped in kernel, `flush_kernel_vmap_range`
      is also called to flush the cpu cache
  - `system_heap_mmap` calls `remap_pfn_range` to map the pages in the
    original `buffer->sg_table` (to userspace vma)
  - `system_heap_vmap` vmaps the pages (to kernel space)
    - `system_heap_do_vmap` calls `vmap` to map all pages in
      `buffer->sg_table` and updates `buffer->vaddr`
  - `system_heap_vunmap` undoes vmap
  - `system_heap_dma_buf_release` frees the buffer
- `dma_heap_init` inits the dma-heap core
  - `alloc_chrdev_region` reserves a `dev_t` major
  - `class_create` creates `dma_heap` class
- `dma_heap_add` creates a dma-heap
  - it reserves a `dev_t` minor from `dma_heap_minors`
  - `cdev_init` and `cdev_add` add a char dev, `heap->heap_cdev`
  - `device_create` creates a `device` tracked in `heap_list`
- `dma_heap_fops` is the fops on the heap char dev
  - `dma_heap_open` looks up the minor in `dma_heap_minors`
  - `dma_heap_ioctl` supports only `DMA_HEAP_IOCTL_ALLOC`
  - `dma_heap_ioctl_allocate` allocates a dma-buf and returns the fd
    - `len` is the alloc size
    - `fd_flags` can only have `O_CLOEXEC` and `O_ACCMODE` (r/w)
    - `heap_flags` must be 0

## udmabuf

- create a dma-buf that wraps ranges of memfds from userspace

## producers and consumers

- producers call `dma_buf_export` to export their buffers as dma-bufs
  - `drivers/accel/habanalabs`
    - ioctl `HL_MEM_OP_EXPORT_DMABUF_FD`
  - `drivers/dma-buf/heaps`
    - ioctl `DMA_HEAP_IOCTL_ALLOC`
  - `drivers/dma-buf/udmabuf`
    - ioctl `UDMABUF_CREATE`
  - `drivers/gpu/drm`
    - ioctl `DRM_IOCTL_PRIME_HANDLE_TO_FD`
  - `drivers/media/common/videobuf2`
    - ioctl `VIDIOC_EXPBUF`
    - ioctl `DMX_EXPBUF`
  - `drivers/misc/fastrpc`
    - ioctl `FASTRPC_IOCTL_ALLOC_DMA_BUFF`
  - `drivers/virtio/virtio_dma_buf`
    - used by drm/virtio
  - `drivers/xen`
    - ioctl `IOCTL_GNTDEV_DMABUF_EXP_FROM_REFS`
- consumers call `dma_buf_attach` to add themselves to the attachment
  lists of dma-bufs
  - `drivers/accel/ivpu` and `drivers/accel/qaic`
    - they are just drm drivers; see below
  - `drivers/gpu/drm`
    - ioctl `DRM_IOCTL_PRIME_FD_TO_HANDLE`
  - `drivers/media/common/videobuf2`
    - ioctl `VIDIOC_QBUF`
  - `drivers/media/platform/nvidia/tegra-vde`
    - ioctl `VIDIOC_*`
  - `drivers/misc/fastrpc`
    - ioctl `FASTRPC_IOCTL_*`
  - `drivers/virtio/virtio_dma_buf`
    - used by drm/virtio
  - `drivers/xen`
    - ioctl `IOCTL_GNTDEV_DMABUF_IMP_TO_REFS`

## `dmabuf` filesystem

- in kernel init, `dma_buf_init` is called
  - it mounts `dmabuf` fs
- when `dma_buf_export` is called to export a driver allocation,
  - it allocates a `struct dma_buf`
  - it allocates a new inode from the fs
  - and it allocates a new file for the inode
