`drm_syncobj`
=============

## History

- an external VkSemaphore cannot be backed by a `sync_file`
  - because vkGetSemaphoreFdKHR wants an fd before there is a `sync_file`
  - it can be backed by BOs instead
- an external VkFence cannot be backed by a `sync_file` or a BO
  - because vkGetFenceFdKHR wants an fd before there is a `sync_file`
  - a local VkFence can be backed by BOs instead
    - there should also be a wait-bo ioctl
  - VkFence allows wait-before-submit.  For a local VkFence, this can be
    accomplished by having three states (unsignaled, pending, signaled) and a
    `cnd_t`.  When a fence is unsignaled, waiting waits on the `cnd_t` first.
  - for an external VkFence, we need kernel support for "external `cnd_t`"
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
