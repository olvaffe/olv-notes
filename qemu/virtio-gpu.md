# Overview

 - qemu provides a virtio-gpu device
   - interface defined in kernel include/uapi/linux/virtio_gpu.h
   - bit VIRTIO_GPU_F_VIRGL decides whether the device supports virgl (3D accel)
   - it is advertised when qemu is compiled with virglrenderer support
 - kernel provides virtio-gpu driver
 - Mesa provides userspace virtio-gpu driver
 - Xorg uses glamor driver
   - uses EGL/GL/GBM
 - 3D buffer allocation
   - userspace does ioctl(DRM_IOCTL_VIRTGPU_RESOURCE_CREATE)
   - kernel sends VIRTIO_GPU_CMD_RESOURCE_CREATE_3D to device
   - qemu calls virgl_renderer_resource_create
   - virglrenderer calls glGenBuffers and glBufferData
 - 3D command buffer
   - opcodes defined in mesa src/gallium/drivers/virgl/virgl_protocol.h
   - mesa builds command buffers and ioctl(DRM_IOCTL_VIRTGPU_EXECBUFFER)
   - kernel sends VIRTIO_GPU_CMD_SUBMIT_3D to device
   - qemu calls virgl_renderer_submit_cmd
   - virglrenderer calls
     - glBlendColor when VIRGL_CCMD_SET_BLEND_COLOR
     - many more

# Memory Management

 - virtio_gpu_resource_create_ioctl
   - every virtio_gpu_object is created with a guest-managed unique
     hw_res_handle (id)
   - guest allocates pages
   - host is also requested to create backing storage (GL buffer)
   - guest calls RESOURCE_ATTACH_BACKING and host creates iovec over guest pages
 - DRM_IOCTL_VIRTGPU_TRANSFER_TO_HOST
   - host copies guest pages into GL buffer
 - DRM_IOCTL_VIRTGPU_TRANSFER_FROM_HOST
   - host copies GL buffer into guest pages
 - DRM_IOCTL_VIRTGPU_MAP
   - map guest pages

# Scanout

* When guest X has a new frame, it either copies the new frame to the scanout
  buffer or makes the new frame the new scanout buffer
  * after copying, guest needs to call `drmModeDirtyFB`
    (`DRM_IOCTL_MODE_DIRTYFB`)
  * to page flip, guest needs to call `drmModeAtomicCommit`
    (`DRM_IOCTL_MODE_ATOMIC`) or `drmModePageFlip`
    (`DRM_IOCTL_MODE_PAGE_FLIP`)
* either way, it triggers a plane update to virtio-gpu kernel driver
  * the driver sends `VIRTIO_GPU_CMD_SET_SCANOUT` to replace the scanout
    buffer
  * the driver sends `VIRTIO_GPU_CMD_RESOURCE_FLUSH` to flush the dirty region
* in qemu, the scanout buffer is a host GL fbo.  It gets blitted to the
  window.
  * set-scanout means to use another fbo for scanout
  * resource-flush means to do the blit and swapbuffer

# Fences

# glxgears

 - after initial setup, interact with the device only in `glXSwapBuffers`
 - specifically, only in the implicit flush
   - `glXSwapBuffers` -> `dri3_swap_buffers` -> ... -> `dri_flush`
 - `dri_flush` waits on the oldest fence associated with the drawable for
   throttling
   - `virgl_fence_finish` -> `virgl_fence_wait` -> `DRM_IOCTL_VIRTGPU_WAIT`
   - `virgl_fence_reference(..., NULL)` -> `virgl_drm_resource_reference` -> `DRM_IOCTL_GEM_CLOSE`
 - `dri_flush` flushes the current context and associate the fence with the
   drawable
   - `virgl_flush_from_st` -> `virgl_drm_winsys_submit_cmd` -> `DRM_IOCTL_VIRTGPU_EXECBUFFER`
     - `_IOC(_IOC_READ|_IOC_WRITE,
             DRM_IOCTL_BASE,
             DRM_COMMAND_BASE + DRM_VIRTGPU_EXECBUFFER,
             sizeof(struct drm_virtgpu_execbuffer))`
     - `_IOC(_IOC_READ|_IOC_WRITE, 0x64, 0x42, 0x20)`
   - `virgl_flush_from_st` -> `virgl_cs_create_fence` -> `DRM_IOCTL_VIRTGPU_RESOURCE_CREATE`
 - a fence is a BO of size 8
