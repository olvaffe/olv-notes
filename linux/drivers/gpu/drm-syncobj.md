`drm_syncobj`
=============

## Background

- a VkSemaphore is
  - initially unsignaled
  - a signal operation makes it signaled
    - according to `vkQueueSubmit`, it is illegal to signal a signaled
      semaphore
  - a wait operation makes it unsignaled
    - it is illegal to wait a semaphore that is unsignaled and has no pending
      signal operation (no wait-before-submit)
    - it is illegal to have multiple waiters
  - an export operation might steal the payload
    - export-before-submit might or might not be allowed
- a VkFence is
  - initially unsignaled or signaled
  - a signal operation makes it signaled
    - according to `vkQueueSubmit`, it is illegal to signal a signaled
      fence or a fence with pending signal operation
  - a wait operation just waits
    - it does not reset the fence state
    - everything is legal, including wait-before-submit
  - an export operation might steal the payload
    - export-before-submit might or might not be allowed
- the bottom line is, GPU signal/wait operations are much more restrictive
  than CPU signal/wait operations
- before external VkSemaphore,
  - a local VkSemaphore can be a dummy BO
    - the BO gets added to EXECBUFFER for explicit waiting and signaling
  - it can be a sync fd that is initially -1
    - a real sync fd is assigned after EXECBUFFER
  - it can also be a simple seqno
    - who gets the seqno from kernel
    - and kernel provides a way to wait for the seqno
- after external VkSemaphore,
  - a VkSemaphore can be a dummy BO
    - it exports an opaque fd (dmabuf of BO)
  - it can be a sync fd
    - it returns the sync fd
    - this works because `vkGetSemaphoreFdKHR` does not allow
      export-before-submit for sync fd
  - it cannot be a simple seqno however
    - unless the seqno is in a shmem that is shared by all
- before external VkFence,
  - a local VkFence can be a dummy BO plus an int plus cv
    - BO is used the same way as in VkSemaphore
    - kernel needs to provide a BO wait iotctl
    - the int should have three states: reset, submitted, signaled
    - wait-before-submit requires a mutex and a cv
  - it can be a sync fd plus an int plus cv
    - just like with dummy BO
  - it can also be a simple seqno
    - same as in VkSemaphore
    - 0 is special and means signaled
    - `UINT64_MAX` is also special and means reset
    - wait-before-submit just works
- after external VkFence,
  - nothing works unless we put int/cv/seqno in a shmem
- we need a new kernel mechanism
- then timeline semaphore comes along
  - initially can be any value
  - a signal operation sets it to any value greater than the current one
    - both CPU and GPU can signal
    - two pending signal operations must have a dependency to make sure the
      value is always monotonically increasing
  - a wait operation waits until it reaches a certain value
    - both CPU and GPU can wait
    - wait-before-submit is allowed and wait can be indefinite
  - it is possible to implement timeline semaphore using binary VkSemaphore
    and VkFence (and mutex and cv)
    - the only thing not possible is external timeline semaphore, which
      requires the states to be on a shmem

## History

- In June 2017, `drm_syncobj` was merged in 4.13 to support external
  VkSemaphore
  - `DRM_CAP_SYNCOBJ` to check support
  - `DRM_IOCTL_SYNCOBJ_CREATE`
    - a `drm_syncobj` has a pointer to a `dma_fence`
      - initially, the pointer is NULL and cannot be waited by execbuffer
      - execbuffer waits dma-fences in in-syncobjs and points out-syncobjs to
        the new `dma_fence`
  - `DRM_IOCTL_SYNCOBJ_DESTROY`
  - `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD`
  - `DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE`
    - exporting/importing allows a syncobj to be shared
    - there are flags to export/import `sync_file` fd instead, with different
      semantics
      - exporting to a `sync_file` returns a `sync_file` for the current
        dma-fence
      - importing a `sync_file` updates the pointer
- In August 2017, `drm_syncobj` was extended in 4.14 to support external
  VkFence
  - `DRM_IOCTL_SYNCOBJ_RESET`
    - points dma-fence pointer to NULL
    - modeled after `vkResetFences`
  - `DRM_IOCTL_SYNCOBJ_SIGNAL`
    - points dma-fence pointer to a special already-signaled dma-fence
    - unused because there is also `DRM_SYNCOBJ_CREATE_SIGNALED` flag?
  - `DRM_IOCTL_SYNCOBJ_WAIT`
    - wait for any or all syncobjs to be signaled with timeout
    - modeled after `vkWaitForFences`
    - `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT` for wait-before-submit behavior
      - it waits until a syncobj gets a dma-fence, and then waits on the fence
- In April 2019, `drm_syncobj` was extended in 5.2 to support external
  timeline VkSemaphore
  - conceptually,
    - a syncobj's dma-fence pointer is replaced by {0, dma-fence}
    - there is a list of {seqno, dma-fence}
  - in practice, `dma_fence_chain` is introduced
  - `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL`
    - to signal a syncobj, add {seqno, already-signaled-stub-fence}
  - `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT`
    - wait for a {seqno, dma-fence} to signal
    - `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT` still works
    - `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_AVAILABLE` waits for the fence to appear,
      but does not wait for the fence to finish
  - `DRM_IOCTL_SYNCOBJ_QUERY`
    - when no flag, return the seqno of the latest point that has signaled
    - when `DRM_SYNCOBJ_QUERY_FLAGS_LAST_SUBMITTED`, when the seqno of the
      latest point
  - `DRM_IOCTL_SYNCOBJ_TRANSFER`
    - convert between binary and timeline syncobjs?
- Driver Support
  - `DRIVER_SYNCOBJ`
    - amdgpu: 4.13
    - i915: 4.13
    - msm: 5.8
  - `DRIVER_SYNCOBJ_TIMELINE`
    - amdgpu: 5.6
    - i915: 5.10

## `struct drm_syncobj`

- `drm_syncobj::fence` is a pointer to a `dma_fence`
  - when the pointer is NULL, the syncobj is unsignaled with no pending signal
    operation
  - when the pointer is non-NULL, the syncobj has the same state
    (signaled/unsignaled) as the fence
  - marking a syncobj signaled from CPU is commonly done via setting the
    pointer to an already-signaled fence
    - `dma_fence_get_stub` is the said fence
- conceptually, a timeline syncobj has a list of {seqno, dma-fence} pairs
  - each pair is a point on the timeline
  - a point is reached when the dma-fence of the point is signaled
  - the dma-fence of a point can be NULL, meaning unsignaled with no pending
    signal operation
  - this extends the traditional binary syncobj
    - the traditional syncobj can be viewed as a timeline syncobj with only
      one point {0, dma-fence}
- in practice, `dma_fence_chain` is introduced
  - a `dma_fence_chain` is itself a `dma_fence`
    - `drm_syncobj::fence` pointer can thus point to a chain
  - a `dma_fence_chain` is also a node in a chain of `dma_fence`
    - internally, it has a pointer to the previous node of the chain
  - a `dma_fence_chain` is signaled when all nodes are signaled
    - see `dma_fence_chain_signaled`
    - it looks like the nodes can signal out-of-order
      - vk does not allow that, but kernel does not care
- a syncobj can switch between a timeline one or a traditional/binary one
  - depending on whether its fence pointer points to a chain or not
  - there is no syncobj "type"

## ioctls

- when a gpu job signals a syncobj, the drm driver assigns the dma-fence
  associated with the job to the syncobj
  - if binary, `drm_syncobj_replace_fence` replaces `syncobj->fence`
  - if timeline, `drm_syncobj_add_point` also replaces `syncobj->fence`
    - the caller must have `dma_fence_chain_alloc`ed a `dma_fence_chain`
    - `dma_fence_chain_init` inits the chain node from `syncobj->fence` and
      the job's fence
      - remember that a chain node has 3 fences
        - `chain->base`, the chain node is itself a fence and the seqno is the
          timeline value
        - `chain->prev` points to the previous node
        - `chain->fence` is the real fence
      - `chain->base` is signaled when both `chain->fence` and `chain->prev`
        are signaled
- `drm_syncobj_signal_ioctl` sets the fence pointer of the syncobj to the
  already-siganled stub fence
- `drm_syncobj_timeline_signal_ioctl` adds {seqno, already-signaled-stub-fence}
  - it calls `drm_syncobj_add_point` which
    - creates a new chain node
    - make `drm_syncobj::fence` the previous node of the new node
    - set `drm_syncobj::fence` to the new node
- `drm_syncobj_reset_ioctl` resets syncobjs
  - it calls `drm_syncobj_replace_fence` to set `syncobj->fence` to NULL
    (unsignaled)
- `drm_syncobj_query_ioctl` queries syncobjs
  - if `syncobj->fence` is a chain (i.e., it is a timeline syncobj),
    - if `DRM_SYNCOBJ_QUERY_FLAGS_LAST_SUBMITTED`, returns the seqno of the
      head chain node
    - otherwise, finds the first unsignaled node and returns its `prev_seqno`
  - if `syncobj->fence` is not a chain (i.e., it is a binary syncobj), returns
    0
- `drm_syncobj_transfer_ioctl`
  - it calls `drm_syncobj_find_fence` on the src syncobj to get the fence
  - it calls `drm_syncobj_add_point` or `drm_syncobj_replace_fence` on the dst
    syncobj to add the fence
  - what does `dma_fence_unwrap_merge` do?

## `syncobj_wait_entry`

- to support `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT`, a syncobj has a list of
  `syncobj_wait_entry`
- A `syncobj_wait_entry` is a wait for a fence to appear.  When `point` is
  set, it waits for a chain node past `point` to appear.
- when there is a change to syncobj fence, `syncobj_wait_syncobj_func` is
  called on all entries to update `fence` field.  When a fence finally
  appears, the waiter is woken up

## `drm_fence_chain`

- it is the latest node of a fence chain
- it is also a subclass of `dma_fence`
- it encapsulates another `dma_fence`
- a fence chain is signaled when all fences encapsulated by all nodes are
  signaled
- normally, each fence chain has its own fence context and all nodes share the
  same context
  - but when a new node is added with an out-of-order seqno, the node and all
    following nodes will be in a different fence context.
- `dma_fence_chain_init` initializes a new chain node
  - if the node is the first in the chain (no `prev` or `prev` is not a chain
    node), it creates a new fence context
  - if the node is added to an existing chain, the new node shares the fence
    context
  - but if the node is out-of-order, the new node starts a new fence context
- in other words,
  - when a `drm_syncobj` is created, depending on whether it is created
    signaled, its `fence` points to a signaled dummy fence or is NULL
  - when the first point is added to make `drm_syncobj` a timeline, a new
    fence chain is initialized and a new fence context is created
    - because `prev_chain` is NULL
  - each timeline `drm_syncobj` has its own fence context
  - the points in the timeline `drm_syncobj` are the seqnos of the fence
    context
- `dma_fence_chain_find_seqno` returns the first node past the specified seqno
- `dma_fence_chain_walk` returns the next node, ignoring and GCing signaled
  nodes
- `dma_fence_chain_for_each` loops over the head node and all unsignaled nodes
  - it uses `dma_fence_chain_walk` internally

## mapping `drm_syncobj` to Vulkan

- creation
  - a binary semaphore is created a `drm_syncobj` with {0, NULL}
  - a fence is created a `drm_syncobj` with {0, NULL} or
    {0, already-signaled-stub-fence}
  - a timeline semaphore is created a `drm_syncobj` with
    {initialValue, already-signaled-stub-fence}
- cpu signal
  - `vkSignalSemaphore` maps to `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL`
- cpu wait
  - `vkWaitForFences` maps to `DRM_IOCTL_SYNCOBJ_WAIT` with
    `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT`.
    - timeout and `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL` are set accordingly
  - `vkWaitSemaphores` maps to `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT` with
    `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT`
    - timeout, `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL`, and points are set
      accordingly
- cpu reset
  - `vkResetFences` maps to `DRM_IOCTL_SYNCOBJ_RESET`
- cpu query
  - `vkGetFenceStatus` maps to `DRM_IOCTL_SYNCOBJ_WAIT` with timeout 0 and
    `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT`
    - return 0 means ready; -ETIME means not ready
  - `vkGetSemaphoreCounterValue` maps to `DRM_IOCTL_SYNCOBJ_QUERY`
- gpu wait/signal
  - `vkQueueSubmit`
    - duplicates all submit info
    - handle wait-before-submit in userspace
      - find all wait semaphores that the driver knows will never signal
      - if any, save the submit info and return
      - a submit thread is spawned to wait in userspace
    - submit
  - for signal, kernel adds or updates {val, dma-fence}
  - for wait, kernel waits val
- export
  - to opaque fd, `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD`
  - to sync fd, set `DRM_SYNCOBJ_HANDLE_TO_FD_FLAGS_EXPORT_SYNC_FILE`
  - do not support exporting timeline to sync fd (what
    `DRM_IOCTL_SYNCOBJ_TRANSFER` is for?)
- import
  - from opaque fd, `DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE`
  - from sync fd, set `DRM_SYNCOBJ_FD_TO_HANDLE_FLAGS_IMPORT_SYNC_FILE`
  - do not support importing timeline from sync fd
    (what `DRM_IOCTL_SYNCOBJ_TRANSFER` is for?)
