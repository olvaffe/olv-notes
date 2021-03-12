QEMU gfxstream
==============

## gfxstream

- protocol
  - <https://android.googlesource.com/device/generic/vulkan-cereal/>
- guest umd
  - <https://android.googlesource.com/device/generic/goldfish-opengl/>
  - umd is at `system/vulkan`
  - encoder is at `system/vulkan_enc`
  - stream is at `system/libOpenglSystemCommon`
  - ASG is at `shared/GoldfishAddressSpace`
- guest kmd
  - <https://android.googlesource.com/kernel/common-modules/virtual-device/>
  - branch android-mainline
  - there are goldfish pipe drivers (old) and ASG driver (new)
  - gfxstream-over-virtio-gpu uses regular virtio-gpu driver
- host renderer
  - <https://android.googlesource.com/platform/external/qemu>
  - branch emu-master-dev
  - device emulation is at `android-qemu2-glue/emulation`
  - real implementation is at `android/android-emu/android/emulation`
  - host renderer is at `android/android-emugl/host/libs/libOpenglRender/vulkan`

## Transport

- when the first Vulkan function is called by a new thread, umd
  - calls `HostConnection::get` to get a `HostConnection`
  - calls `HostConnection::rcEncoder` to get a `ExtendedRCEncoderContext`
  - calls `ResourceTracker::get` to set up `ResourceTracker`
  - calls `HostConnection::vkEncoder` to get a `VkEncoder`
- `HostConnection::connect`
  - the transport of interests are
    - `HOST_CONNECTION_ADDRESS_SPACE`
    - `HOST_CONNECTION_VIRTIO_GPU_ADDRESS_SPACE`
  - calls `createAddressSpaceStream` or `createVirtioGpuAddressSpaceStream` to
    create a `AddressSpaceStream`
- on virtio-gpu, these ioctls are used
  - `DRM_IOCTL_VIRTGPU_RESOURCE_CREATE`
    - for `virtgpu_address_space_create_context_with_subdevice`
    - for gbm allocations
  - `DRM_IOCTL_VIRTGPU_RESOURCE_CREATE_BLOB`
    - for `virtgpu_address_space_allocate_hostmem`
    - for `ResourceTracker::Impl::on_vkAllocateMemory`
  - `DRM_IOCTL_VIRTGPU_EXECBUFFER`
    - for ASG ioctls
  - `DRM_IOCTL_VIRTGPU_WAIT`
    - for `virtgpu_address_space_ping_with_response`
- `AddressSpaceStream` is an `IOStream`
  - `writeFully` writes a type3 command and waits for it to finish
    - `ring_buffer_view_write` writes to the type3 ring
      - it writes partially when there is not enough space
      - if none is written, it busy waits (with exponential backoff)
    - if ring is idle, it pings
    - unless `writeFullyAsync`, it busy waits (with exponential backoff) until
      the data is consumed
  - `allocBuffer` and `commitBuffer` writes a type1 command
    - `allocBuffer` returns the type1 ring tail, busy waits if necessary
    - `commitBuffer` increments tail
  - `readFully` reads the data, busy waits (with exponential backoff) if
    necessary
- `VulkanStreamGuest` is used to wrap an `IOStream`

## Vulkan Commands

- when a `vkGet*` is called, such as `vkGetPhysicalDeviceFeatures`,
  - it jumps to `entry_vkGetPhysicalDeviceFeatures` first
  - the entrypoint gets the thread-local encoder and calls
    `VkEncoder::vkGetPhysicalDeviceFeatures`
  - it reserves from the stream, which waits until the ring buffer has enough
    space
  - it marshals to the ring buffer
  - unmarshaling calls `IOStream::readback` which implies `commit` to update
    ring tail
- when a `vkCreate*` is called, such as `vkCreateBuffer`,
  - it jumps to `entry_vkCreateBuffer` first
  - the entrypoint calls the global `ResourceTracker`'s
    `ResourceTracker::on_vkCreateBuffer`
  - it calls `VkEncoder::vkCreateBuffer`
    - which additionally sets up "handle mapping"
    - which reads the `VkResult` so `vkCreateBuffer` is sync
  - the buffer is added to `info_VkBuffer`, which is an `std::unordered_map`,
    for tracking
- when a `vkDestroy*` is called, such as `vkDestroyBuffer`,
  - it sets up "destroy mapping"
  - it only flushes so `vkDestroyBuffer` is async

## VkDeviceMemory

- `vkAllocateMemotry` calls `entry_vkAllocateMemory` calls
  `ResourceTracker::on_vkAllocateMemory`
- it does a lot
- `getOrAllocateHostMemBlockLocked` calls
  - `DRM_IOCTL_VIRTGPU_RESOURCE_CREATE_BLOB`
  - `DRM_IOCTL_VIRTGPU_MAP`

## VkFence

- assume `hasVirtioGpuNativeSync` is true
- `ResourceTracker::on_vkCreateFence`
  - calls `VkEncoder::vkCreateFence` without export info
- `ResourceTracker::on_vkGetFenceFdKHR`
  - sends `VIRTIO_GPU_NATIVE_SYNC_VULKAN_CREATE_EXPORT_FD` via
    `DRM_IOCTL_VIRTGPU_EXECBUFFER` with
    `VIRTGPU_EXECBUF_FENCE_FD_OUT`
  - the command is named `kVirtioGpuNativeSyncVulkanCreateExportFd` in the
    host
    - it seems the fence handle is ignored
    - I guess all queue operations are automatically fenced by the host??
- `ResourceTracker::on_vkImportFenceFdKHR`
  - take the sync fd
- `ResourceTracker::on_vkWaitForFences`
  - if all fences are internal, call host `vkWaitForFences` synchronously
  - if any fence is external, use a thread pool to wait on individual fences

## VkSemaphore

- assume `hasVirtioGpuNativeSync` is true
- `ResourceTracker::on_vkCreateSemaphore`
  - calls `VkEncoder::vkCreateSemaphore`
    - export info is not passed when exporting sync fd
  - sends `VIRTIO_GPU_NATIVE_SYNC_VULKAN_CREATE_EXPORT_FD` via
    `DRM_IOCTL_VIRTGPU_EXECBUFFER` with
    `VIRTGPU_EXECBUF_FENCE_FD_OUT`
    - it works on both fences and semaphores?
    - it seems the handle is ignored
- `ResourceTracker::on_vkGetSemaphoreFdKHR`
  - if sync fd, dup it
  - if opaque fd, uses a memfd to remember the host fd val
- `ResourceTracker::on_vkImportSemaphoreFdKHR`
  - if sync fd, take it
  - if opaque fd, read back the host fd val and calls
    `VkEncoder::vkImportSemaphoreFdKHR`
- `ResourceTracker::on_vkQueueSubmit`
  - if all wait semaphores are internal, do nothing
  - otherwise, do a guest side wait using a thread pool
