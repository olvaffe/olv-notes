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
  - initialized unsignaled or signaled
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

## History

- In June 2017, `drm_syncobj` was merged to support external VkSemaphore with
  only
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
- In August 2017, `drm_syncobj` was extended to support external VkFence with
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
- In April 2019, `drm_syncobj` was extended to support external timeline
  VkSemaphore with
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
- `dma_fence_chain_find_seqno` returns the first node past the specified seqno
- `dma_fence_chain_walk` returns the next node, ignoring and GCing signaled
  nodes

## timeline syncobj

- conceptually,
  - a syncobj used to have a pointer to a dma-fence
  - now it has a list of {seqno, dma-fence} as points on a timeline
  - the dma-fence in the legacy view becomes {0, dma-fence}
  - a point is signaled when the associated dma-fence is signaled
  - a syncobj is signaled when all points are signaled
  - in practice, the pointer in syncobj now points to a `dma_fence_chain`
- `drm_syncobj_add_point` adds a new point
  - in practice, it adds a new chain node to the previous node and
    replaces the syncobj pointer to point to the new node
- `drm_syncobj_timeline_signal_ioctl` adds an already signaled point
  {seqno, already-signaled-stub-fence} 
