virgl
=====

## Status

- OpenGL 4.3
- OpenGL ES 3.1

## Gallium Winsys

- `transfer_get` is `VIRTGPU_TRANSFER_FROM_HOST`
- `transfer_put` is `VIRTGPU_TRANSFER_TO_HOST`
- `resource_create` is `VIRTGPU_RESOURCE_CREATE` with reuse
  - for VBO, UBO, IBO, or custom BO (for fences), attemp to reuse
  - reuse check requires a `VIRTGPU_WAIT` to check for busy
- `resource_unref` is `GEM_CLOSE` with reuse and refcount
- `resource_map` is `VIRTGPU_MAP` followed by `mmap`
- `resource_wait` is `VIRTGPU_WAIT`
- `cmd_buf_create` allocates a struct on heap
- `emit_res` tracks a BO in cmdbuf
- `res_is_referenced` checks if a BO is used by cmdbuf
  - dummpy impl and almost always returns true at the moment
- `submit_cmd` is `VIRTGPU_EXECBUFFER`
- `cs_create_fence` creates a dummy resource, optionally with an addional
  `sync_file`
  - it reuses; does that work?
- `fence_wait` is `VIRTGPU_WAIT` on the dummy resource, optionally
  `sync_wait` as well
- `fence_server_sync` is `sync_accumulate`, accumulate optional `sync_file`
  to the cmdbuf
- `fence_get_fd` gets the optional `sync_file`

## Gallium Driver

- resource `clean_mask` and `virgl_resource_dirty`
  - a (level of a) resource is clean when the guest sees up-to-date contents
    of the resource (i.e., no host write since last sync)

## Kernel Init

- `virtio_gpu_probe` calls `virtio_gpu_init`
- `virtio_gpu_init` checks device features
  - `VIRTIO_F_VERSION_1` to require virtio-gpu v1
  - `VIRTIO_GPU_F_VIRGL` for virgl
  - `VIRTIO_GPU_F_EDID` for EDID
- it calls `virtio_find_vqs` to initialize two `virtqueue`s, control and
  cursor
- it calls `virtio_gpu_ttm_init` to initialize TTM
- it calls `virtio_gpu_modeset_init` to initialize the KMS
- it calls `virtio_gpu_get_capsets` to initialize capsets
  - e.g., capset id 2 corresponds to `virgl_caps_v2`
  - used for answering `PIPE_CAP_xxx` queries
- it calls `virtio_gpu_cmd_get_edids` to initialize EDID
- it calls `virtio_gpu_cmd_get_display_info` to initialize display info
  - display modes available

## Kernel Capsets

- virtio-gpu pci config space (`virtio_gpu_config::num_capsets`) specifies the
  capset count
- `virtio_gpu_cmd_get_capset_info` is called for each capset to set up `id`,
  `max_version`, and `max_size` for each `virtio_gpu_drv_capset`
- `virtio_gpu_get_caps_ioctl` is called by the userspace
  - userspace specifies `cap_set_id`, `cap_set_ver`, and `size`
  - `cap_set_id` must match
  - `cap_set_ver` must be less than or equal to `max_version`
  - `size` is capped to `max_size`
  - `virtio_gpu_cmd_get_capset` is called to query capset id/ver with
    `max_size`
- because `max_size` can grow, version is bumped when there are
  backward-incompatible changes
- `VIRTGPU_PARAM_CAPSET_QUERY_FIX`
  - without it, kernel always copies `max_size` to userspace
  - that is problematic because userspace does not know `max_size`
- capset id 1 is `VIRTIO_GPU_CAPSET_VIRGL`
  - `max_size` is fixed and is known by userspace
  - `max_version` is always 1 but userspace always asks for 0
- capset id 2 is `VIRTIO_GPU_CAPSET_VIRGL2`
  - `max_size` can change and is unknown by userspace
  - `max_version` is always 2 but userspace always asks for 0

## Kernel virtqueue

- a queue is a ring buffer
  - both the kernel and the device have rw access
  - the kernel writes using `virtqueue_add_sgs` and notifies the device using
    `virtqueue_kick`
  - after the device writes, the associated `vq_callback_t` is called.  The
    callback function normally schedules a work for later execution
- `virtio_gpu_get_vbuf` allocates a command packet, `virtio_gpu_vbuffer`
  - allocated using `vgdev->vbufs` slab
  - each allocation takes `sizeof(virtio_gpu_vbuffer) + 120`
    - 96 bytes for inline cmd
    - 24 bytes for inline response
  - must specify response buffer and response callback
  - there is also `data_buf`, where data are things like sg table for BOs, or
    cmdbufs for execbuf
- `virtio_gpu_queue_ctrl_buffer` queues a command packet
- after the device writes, `virtio_gpu_ctrl_ack` is called to schedule
  `virtio_gpu_dequeue_ctrl_func`
  - the scheduled function disables callback temporarily to process the
    responses
  - if there is a response callback, call it
  - free the command packet
  - retire some fences

## Kernel Fences

- `virtio_gpu_queue_fenced_ctrl_buffer` queues a command packet and returns a
  `virtio_gpu_fence`.  The fence is signaled after the command packet is
  executed.
- It works by
  - init fence to the next seqno, `fence->seq = ++drv->sync_seq`
  - add fence to `drv->fences`
  - mark the cmd with `VIRTIO_GPU_FLAG_FENCE` and the seqno
- after the command is executed, `virtio_gpu_dequeue_ctrl_func`
  - get the largest seqno from all executed commands
  - call `virtio_gpu_fence_event_process` to retire all `drv->fences` whose
    `fence->seq` is smaller or equal
  - save the seqno to `vgdev->fence_drv.last_seq` for DRM fences
- DRM fences
  - each fence is also a DRM fence
  - the lifetime of the fence may be prolonged
  - status can be checked by comparing `fence->seq` and
    `vgdev->fence_drv.last_seq`

## Kernel BOs

- `VIRTGPU_WAIT` waits for a BO to be ready
  - calls `ttm_bo_reserve` to lock the reservation object
  - calls `ttm_bo_wait` to wait on all fences, exclusive or shared
  - calls `ttm_bo_unreserve` to unlock the reservation object
- `VIRTGPU_TRANSFER_FROM_HOST`
  - calls `ttm_bo_reserve` to lock the reservation object
  - calls `ttm_bo_validate` to move the BO to the right heap
  - calls `virtio_gpu_cmd_transfer_from_host_3d` to queue a DMA
  - calls `reservation_object_add_excl_fence` to add the DMA fence 
  - calls `ttm_bo_unreserve` to unlock the reservation object
- `VIRTGPU_TRANSFER_TO_HOST`
  - similar to `VIRTGPU_TRANSFER_FROM_HOST`
- `VIRTGPU_RESOURCE_CREATE` creates a BO
  - `virtio_gpu_alloc_object` to initialize a `virtio_gpu_object`
    - it has the regular GEM backing store (shmem)
  - `virtio_gpu_object_list_validate` to move the BO to the right heap
  - `virtio_gpu_cmd_resource_create_3d` to queue a resource create cmd
  - `virtio_gpu_object_attach` to set up an sg table and to queue an attach
    cmd that sends the sg table to the device
  - `ttm_eu_fence_buffer_objects` to make the cmd fence a exclusive fence of
    the BO
- `VIRTGPU_MAP` returns an offset for `mmap`ing a BO
- `VIRTGPU_RESOURCE_INFO` to query the resource id/size (after importing a
  dma-buf)
- `VIRTGPU_EXECBUFFER`
  - calls `virtio_gpu_object_list_validate` to validate all referenced BOs
    - it reserves the BOs
    - it also validates the BOs, making sure they are allocated and on the
      right heaps
  - queues a `VIRTIO_GPU_CMD_SUBMIT_3D` command packet to execute the cmdbuf
  - calls `ttm_eu_fence_buffer_objects` to make the cmd fence a exclusive
    fence of the referenced BOs

## Kernel BO Reference Counting

- `DRM_IOCTL_VIRTGPU_RESOURCE_CREATE`
  - `drm_gem_object_init` sets refcount to 1
  - `virtio_gpu_object_create` gets and puts when fenced, refcount 1
  - `drm_gem_handle_create` gets for handle, refcount 2
  - `virtio_gpu_resource_create_ioctl` explicitly
    `drm_gem_object_put_unlocked` before returning the handle, refcount 1
- `DRM_IOCTL_GEM_CLOSE` (with no other reference)
  - `virtio_gpu_gem_object_close` prepares for close and has no effect on
    refcount, refcount 1
  - `drm_gem_object_release_handle` releases the handle and puts, refcount 0
  - `virtio_gpu_gem_free_object` puts the contained ttm bo, which also becomes
    refcount 0
  - `virtio_gpu_ttm_bo_destroy` frees the bo with the contained ttm bo
- insert `DRM_IOCTL_VIRTGPU_TRANSFER_TO_HOST` before close?
  - `drm_gem_object_lookup` on enter
  - `drm_gem_object_put_unlocked` on exit
  - no difference
- the problem is TTM
  - `ttm_bo_put` delays destroy when the BO is busy
    (`ttm_bo_cleanup_refs_or_queue`) and adds BO to an delete list
  - `ttm_bo_delayed_delete` wakes up every HZ/100 (10ms) and really destroys a
    BO when it is idle (`ttm_bo_cleanup_refs`)

## Kernel SHMEM helper

- use `DEFINE_DRM_GEM_SHMEM_FOPS` to define the `file_operations`
- there should be an ioctl to create a BO
  - the BO should embed a `drm_gem_shmem_object`, which is a `drm_gem_object`
- there should be an ioctl to return a magic offset for mapping a BO
  - return `drm_vma_node_offset_addr` from the BO
- when the DRM file is mapped with the magic offset, `drm_gem_shmem_mmap` is
  called
  - it calls `drm_gem_mmap` to look up the BO from the magic offset
  - `vm_area_struct` is set up with
    - `VM_IO`
    - `VM_MIXEDMAP`
    - `pgprot_writecombine`
    - `drm_gem_shmem_vm_ops`
  - pagefault calls `vmf_insert_page`
- `vma->vm_flags` is set to `0x140440fb`
  - `VM_SHARED`, `VM_WRITE`, `VM_READ`
  - `VM_MAYREAD`, `VM_MAYWRITE`, `VM_MAYEXEC`, `VM_MAYSHARE`
  - `VM_IO`, `VM_DONTEXPAND`, `VM_DONTDUMP`
  - `VM_MIXEDMAP`
  - `vm_get_page_prot` maps it to `PAGE_SHARED`
    - `_PAGE_PRESENT`
    - `_PAGE_RW`
    - `_PAGE_USER`
    - `_PAGE_ACCESSED`
    - `_PAGE_NX`
- `vma->vm_page_prot` is set to `0x800000000000002f` on x86
  - PAT and PCD bits are cleared and PWT bit is set.
  - according to `pat_init`, it maps to `_PAGE_CACHE_MODE_WC`

## Kernel Misc ioctls

- `VIRTGPU_GET_CAPS` queries Gallium caps
- `VIRTGPU_GETPARAM` queries driver/device params

## virtio-gpu Misc Commands

- `VIRTIO_GPU_CMD_GET_CAPSET_INFO`
- `VIRTIO_GPU_CMD_GET_EDID`

## Display

- there is no vblank support; pageflip is executed immediately
  - vblank event of type page-flip-complete is still delivered
- when pageflip happens, a set scanout command and a resource flush command
  are queued
- when the user space decides the current fb is dirty and does a DIRTYFB
  ioctl, a resource flush command is queued
- qemu
  - there is a host win fb and a guest fb
  - upon set scanout command, it calls `dpy_gl_scanout_texture` to make the
    buffer the guest fb
  - upon resource flush command, it calls `dpy_gl_update` to blit guest fb to
    host win fb

## QEMU virtio-gpu

* `pci_vga_init`
  * `virtio_pci_realize`
  * `virtio_vga_base_realize`
    * add BAR0, 8MB, `vga.vram`
  * `virtio_pci_device_plugged`
    * add BAR2, 16KB, `virtio-pci`
      * offset  0KB, size 4KB: COMMON CFG
      * offset  4KB, size 4KB: ISR CFG
      * offset  8KB, size 4KB: DEVICE CFG
      * offset 12KB, size 4KB: NOTIFY CFG
    * add BAR4, 4KB, `virtio-vga-msix`
* after the guest writes to the vq, it writes to NOTIFY CFG to notify the host
  * the host handles the writes and call `virtio_queue_notify`
  * it calls `virtio_gpu_handle_ctrl` eventually
  * to see this in ftrace,
    * `lspci -vvnn` in guest to find the gpa of BAR2
    * add 0x3000 to the gpa
    * ftrace `kvm_mmio`
* after the host writes to the vq, it calls `virtio_notify`
  * it ends up in `virtio_pci_notify`
  * to see this in ftrace,
    * `lspci -vvnn` in guest to find the IRQ of the device
    * ftrace `kvm_set_irq`

## virglrenderer

- `virgl_renderer_init`
  - initializes the global `vrend_state` of type `global_renderer_state`
  - creates the global resource hash table, `res_hash`
  - creates a temporary gl context to get caps and then destroys it
  - create context 0, and sub-context 0 in it, for internal use
  - contexts are stored in the global `dec_ctx` array of type
    `vrend_decode_ctx`
- `virgl_renderer_context_create`
  - creates a client context of type `vrend_context` with context id >0
    - in gallium driver, this maps to the "pipe screen"
  - also creates a `vrend_sub_context` of id 0 for internal use
    - each `vrend_sub_context` has its own gl context
- `virgl_renderer_submit_cmd`
  - parses and executes the `VIRGL_CCMD_*` commands
    - `VIRGL_CCMD_CREATE_SUB_CTX` create a client sub-context, which maps to a
      "pipe context" in the gallium driver
    - `VIRGL_CCMD_SET_SUB_CTX` selects a sub-context and makes it current
    - `VIRGL_CCMD_CREATE_OBJECT` adds a "pipe state" to the current sub-context
- `virgl_renderer_resource_create`
  - creates a `vrend_resource` and inserts it to the global `res_hash`
  - a `vrend_resource` owns a GL resource
- `virgl_renderer_ctx_attach_resource`
  - makes a resource available to a context (i.e., add it to the context's
    resource hash table)
  - `VIRGL_CCMD_*` can only reference attached resources
- `virgl_renderer_resource_attach_iov`
  - attaches iovs to a resource for transfers
- `virgl_renderer_transfer_write_iov`
  - copies data from the attached iovs to the GL resource of a
    `vrend_resource`
- `virgl_renderer_create_fence`
  - inserts a fence into the GL command stream
- `virgl_renderer_poll`
  - check the status of the inserted fences
  - if a fence is signaled, notify the caller of its associated seqno

## virglrenderer: vtest

- normal operation
  - listen to `/tmp/.virgl_test` and wait in
    `vtest_main_wait_for_socket_accept`
  - for each new connection, fork a child to run `vtest_main_run_renderer`
  - `vtest_main_run_renderer` will
    - poll for input
    - read vtest header
    - `virgl_renderer_poll` to update fence seqno
    - dispatch to using the `vtest_commands` dispatch table
    - `virgl_renderer_create_fence` to add a new fence
- a client must send `VCMD_CREATE_RENDERER` first
  - it initializes the global variable `vtest_renderer`
  - it calls `virgl_renderer_context_create` with context id 1
- `VCMD_SUBMIT_CMD` calls `virgl_renderer_submit_cmd`
- `VCMD_RESOURCE_CREATE2`
  - calls `virgl_renderer_resource_create` first
  - calls `virgl_renderer_ctx_attach_resource` to attach the resource to
    the context
  - creates a memfd, mmaps it, and sends the fd to the client
  - calls `virgl_renderer_resource_attach_iov` to attach the iov to the
    resource 
- `VCMD_TRANSFER_PUT2`
  - calls `virgl_renderer_transfer_write_iov` to copy from the attached memfd
    iov to the resource
