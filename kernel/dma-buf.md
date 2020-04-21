# DMA-BUF

## dma-fence

* `struct dma_fence` has
  * `rcu` for use with `kfree_rcu`
  * `cb_list` for a list of callbacks to be invoked when the fence signals
  * `lock` for a spinlock that must be held to access some parts of the fence
  * `context/seqno` for the seqno of the fence in the context
  * `flags` for its internal status (signal enabled, signaled, timestamped)
  * `timestamp` to indicate when it was signaled
  * `error` to indicate that the fence is signaled because of errors
* dma-fence is designed to be RCU-protected.  A writer can `dma_fence_put`
  without `synchronize_rcu`.  This is because when the last reference is
  dropped, the fence is freed with `kfree_rcu`
  * on the other hand, a reader might get a fence with refcount==0.  It must
    call `dma_fence_get_rcu` to make sure the fence is usable.
* dma-fence is not signaled timely by default.  To get that behavior, which
  requires enabling IRQ, one of these should be called
  * `dma_fence_add_callback`
  * `->wait`
  * `dma_fence_enable_sw_signaling`
* It usually goes like
  * someone waits for the fence
    * the fence is locked
    * if fence is already signaled, return early
    * otherwise, timely signaling is enabled
    * a callback is added to wake up this process
    * the fence is unlocked
    * the process goes to sleep
  * HW IRQ
    * irq handler calls `dma_fence_signal`
    * callbacks are invoked; usually there is only one which wakes up the
      waiting process
* Only these are protected by the spinlock, as far as I can tell
  * `cb_list`
  * `->enable_signaling`
* `dma_fence_array`
  * it is a fence which consists of a fixed-size array of fences
    * the fence may be on a different timeline because the underlying fences
      may be from different timelines
  * this is used by `sync_file`
* `dma_fence_chain`
  * it is a fence that wraps another fence and supports chaining
  * this is used by timeline syncobj
    * the syncobj is itself a timeline (rather a point on another timeline)
    * the timeline consists of discrete fences, which are points on the
      timeline, and are managed with a dma-fence-chain
    * userspace can
      * userspace can advance the seqno of the timeline
        * appends a signaled fence with the desired seqno to the chain
      * userspace can query the seqnos of the points
        * get seqnos of the fences
      * userspace can wait until the timeline reaches a certain seqno
        * wait for fences before the seqno to signal

## sync-file

* a `struct file` backed by anonymous inode that wraps a dma-fence
* it enables userspace to operate on the dma-fence using `poll`, `ioctl`, as
  well as transferring the dma-fence to another process
* it is possible to create a sync-file by merging two sync-files.  The new
  sync-file wraps a dma-fence-array.

## reservation object

* A `reservation_object` manages fences for a buffer
  * there is an exclusive fence, `fence_excl`
    * this is for fence that writes to the buffer
  * there is also a list of shared fences, `fence`
    * this is for fences that read from the buffer
  * when submitting cmdbuf for GPUs, there are usually many read-only buffers
    and several read-write buffers.  We need to lock all of their resv objs
    such that we can add the dma-fence for the submission to them.  The most
    natural lock type is `ww_mutex`
  * There is also a seqcount.  Together with the mutex for writers, they
    provide a RW lock to protect `obj->fence_excl` and `obj->fence` for
    readers.
  * Each dma-fence is RCU-protected to allow lockfree reads

## dma-buf

* Introduction
  * A device driver can export its DMA buffer as a dma-buf by calling
    `dma_buf_export`
  * The dma-buf can be passed to userspace as a fd by calling `dma_buf_fd`
  * userspace can share this fd with another process/device
  * the other end can call `dma_buf_get` to get the dma-buf back and call
    `dma_buf_attach` to add itself to the dma-buf's attachment list
  * the other end can get the `sg_table` for DMA access to the dma-buf by
    calling `dma_buf_map_attachment` and `dma_buf_unmap_attachment`
  * when done with the dma-buf, `dma_buf_detach` and `dma_buf_put` should be
    called
* `dma_buf_export` create a `struct dma_buf`
  * there is always a `struct reservation_object` for implicit fencing
  * this creates a `struct file` whose `private_data` points back to the
    dma-buf
  * there is `priv` which normally points back to the device driver buffer
    object
* userspace can
  * `lseek` on the fd to get the buffer size
  * `poll` the fd
    * readable when the exclusive fence is signaled
    * writable when all fences are signaled
    * this indicates the DMA tranfers are complete.  Cache flushing might be
      required before CPU access.
  * `mmap` the fd to map the DMA buffer; no synchronization
  * `DMA_BUF_SET_NAME` to assign a name for debugging
  * `DMA_BUF_IOCTL_SYNC` to sync for CPU access
    * `DMA_BUF_SYNC_START` start CPU access
      * usually involves waiting and cache flushing
    * `DMA_BUF_SYNC_READ` / `DMA_BUF_SYNC_WRITE` sync for cpu read / write
      * controls waiting on only exclusive fence or all fences
    * `DMA_BUF_SYNC_END` end CPU access
      * usually involves cache flushing
* `dma_buf_get` and `dma_buf_attach`
  * the other driver is given an fd
  * calls `dma_buf_get` to map it to the dma-buf
  * calls `dma_buf_attach` to notify the original driver the intention to use
  * creates a other-driver-specific buffer object that wraps the dma-buf
* `dma_buf_map_attachment`
  * the other driver calls `dma_buf_map_attachment` to get the `sg_table` before
    DMA
  * the original driver duplicates its `sg_table` for the dma-buf and calls
    `dma_map_sg` to get the dma addresses for the pages
  * this other driver wraps its buffer object around the `sg_table`
* `dma_buf_unmap_attachment` is the opposite of mapping
* `dma_buf_detach`
* dma-buf provides other functions for kernel space CPU fallback
  * `dma_buf_begin_cpu_access` / `dma_buf_end_cpu_access`
  * `dma_buf_kmap` / `dma_buf_kunmap`
  * `dma_buf_vmap` / `dma_buf_vunmap`

## udmabuf

* create a dma-buf that wraps ranges of memfds from userspace

## `dmabuf` filesystem

- in kernel init, `dma_buf_init` is called
  - it mounts `dmabuf` fs
- when `dma_buf_export` is called to export a driver allocation,
  - it allocates a `struct dma_buf`
  - it allocates a new inode from the fs
  - and it allocates a new file for the inode
