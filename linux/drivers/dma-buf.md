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
  - `rcu` for use with `kfree_rcu`
  - `cb_list` for a list of callbacks to be invoked when the fence signals
  - `lock` for a spinlock that must be held to access some parts of the fence
  - `context/seqno` for the seqno of the fence in the context
  - `flags` for its internal status (signal enabled, signaled, timestamped)
  - `timestamp` to indicate when it was signaled
  - `error` to indicate that the fence is signaled because of errors
- dma-fence is designed to be RCU-protected.  A writer can `dma_fence_put`
  without `synchronize_rcu`.  This is because when the last reference is
  dropped, the fence is freed with `kfree_rcu`
  - on the other hand, a reader might get a fence with refcount==0.  It must
    call `dma_fence_get_rcu` to make sure the fence is usable.
- dma-fence is not signaled timely by default.  To get that behavior, which
  requires enabling IRQ, one of these should be called
  - `dma_fence_add_callback`
  - `->wait`
  - `dma_fence_enable_sw_signaling`
- It usually goes like
  - someone waits for the fence
    - the fence is locked
    - if fence is already signaled, return early
    - otherwise, timely signaling is enabled
    - a callback is added to wake up this process
    - the fence is unlocked
    - the process goes to sleep
  - HW IRQ
    - irq handler calls `dma_fence_signal`
    - callbacks are invoked; usually there is only one which wakes up the
      waiting process
- Only these are protected by the spinlock, as far as I can tell
  - `cb_list`
  - `->enable_signaling`
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

## dma-resv

- A `dma_resv` manages fences for a buffer
  - there is an exclusive fence, `fence_excl`
    - this is for fence that writes to the buffer
  - there is also a list of shared fences, `fence`
    - this is for fences that read from the buffer
  - when submitting cmdbuf for GPUs, there are usually many read-only buffers
    and several read-write buffers.  We need to lock all of their resv objs
    such that we can add the dma-fence for the submission to them.  The most
    natural lock type is `ww_mutex`
  - There is also a seqcount.  Together with the mutex for writers, they
    provide a RW lock to protect `obj->fence_excl` and `obj->fence` for
    readers.
  - Each dma-fence is RCU-protected to allow lockfree reads

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

## udmabuf

- create a dma-buf that wraps ranges of memfds from userspace

## `dmabuf` filesystem

- in kernel init, `dma_buf_init` is called
  - it mounts `dmabuf` fs
- when `dma_buf_export` is called to export a driver allocation,
  - it allocates a `struct dma_buf`
  - it allocates a new inode from the fs
  - and it allocates a new file for the inode
