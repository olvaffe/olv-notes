# Initialization

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

# virtqueue

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

# Fences

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

# BOs

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

# Misc ioctls

 - `VIRTGPU_GET_CAPS` queries Gallium caps
 - `VIRTGPU_GETPARAM` queries driver/device params

# Misc Commands

 - `VIRTIO_GPU_CMD_GET_CAPSET_INFO`
 - `VIRTIO_GPU_CMD_GET_EDID`

# Gallium Winsys

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

# Gallium Driver

 - resource `clean_mask` and `virgl_resource_dirty`
   - a (level of a) resource is clean when the guest sees up-to-date contents
     of the resource (i.e., no host write since last sync)
