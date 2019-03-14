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
   - there is also `data_buf`
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

# IOCTLs

 - `VIRTGPU_MAP` returns an offset for `mmap`ing the BO
 -

# Commands

 - `VIRTIO_GPU_CMD_GET_CAPSET_INFO`
 - `VIRTIO_GPU_CMD_GET_EDID`
