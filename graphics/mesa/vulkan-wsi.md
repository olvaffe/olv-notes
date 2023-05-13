Mesa Vulkan WSI
===============

## Overview

- it provides most WSI entrypoints
- `VK_KHR_wayland_surface`
  - `vkCreateWaylandSurfaceKHR`
    - `wsi_CreateWaylandSurfaceKHR`
    - allocates and initializes `wsi_wl_surface`, which inherits from
      `VkIcdSurfaceWayland` as required by the loader
  - `vkGetPhysicalDeviceWaylandPresentationSupportKHR`
    - `wsi_GetPhysicalDeviceWaylandPresentationSupportKHR`
    - calls `wsi_wl_display_init` to check if `zwp_linux_dmabuf_v1` v3 is
      supported
      - or `wl_shm` if swrast
- `VK_KHR_xcb_surface`
  - `vkCreateXcbSurfaceKHR`
    - `wsi_CreateXcbSurfaceKHR`
    - allocates and initializes `VkIcdSurfaceXcb` as required by the loader
  - `vkGetPhysicalDeviceXcbPresentationSupportKHR`
    - `wsi_GetPhysicalDeviceXcbPresentationSupportKHR`
    - calls `wsi_x11_check_for_dri3` to check if DRI3 is supported
      - skipped if swrast
    - calls `visual_supported` to check if the visual is
      `XCB_VISUAL_CLASS_TRUE_COLOR` or `XCB_VISUAL_CLASS_DIRECT_COLOR`
- `VK_KHR_surface`
  - `vkDestroySurfaceKHR`
    - `wsi_DestroySurfaceKHR`
    - if wayland, calls `wsi_wl_surface_destroy`
    - free the struct
  - `vkGetPhysicalDeviceSurfaceSupportKHR`
    - `wsi_GetPhysicalDeviceSurfaceSupportKHR`
    - `wsi_wl_surface_get_support` always returns true on wayland
    - `x11_surface_get_support` performs the same check as
      `wsi_GetPhysicalDeviceXcbPresentationSupportKHR` does
  - `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`
    - `wsi_GetPhysicalDeviceSurfaceCapabilitiesKHR`
    - `wsi_wl_surface_get_capabilities2`
      - current/min/max extents are set `UINT32_MAX`/1x1/gpu max extent
      - `minImageCount` is determined by `wsi_wl_surface_get_min_image_count`
        - it returns 2 or 4
    - `x11_surface_get_capabilities2`
      - current/min/max extents are set to the current window geometry
      - `minImageCount` is determined by `x11_get_min_image_count` or
        `x11_get_min_image_count_for_present_mode`
        - they return 3, 4, or 5
    - in both cases, the images support these usage flags
      - `VK_IMAGE_USAGE_TRANSFER_SRC_BIT`
      - `VK_IMAGE_USAGE_TRANSFER_DST_BIT`
      - `VK_IMAGE_USAGE_SAMPLED_BIT`
      - `VK_IMAGE_USAGE_STORAGE_BIT`
      - `VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT`
      - `VK_IMAGE_USAGE_INPUT_ATTACHMENT_BIT`
  - `vkGetPhysicalDeviceSurfaceFormatsKHR`
    - `wsi_GetPhysicalDeviceSurfaceFormatsKHR`
    - `wsi_wl_surface_get_formats` returns formats advertised by
      `zwp_linux_dmabuf_v1` or `wl_shm`
    - `x11_surface_get_formats` returnsformats compatible with the visual
      - `VK_FORMAT_R5G6B5_UNORM_PACK16` for depth 16
      - `VK_FORMAT_B8G8R8A8_SRGB` and `VK_FORMAT_B8G8R8A8_UNORM` for depth 24
      - `VK_FORMAT_A2R10G10B10_UNORM_PACK32` for depth 30
  - `vkGetPhysicalDeviceSurfacePresentModesKHR`
    - `wsi_GetPhysicalDeviceSurfacePresentModesKHR`
    - `wsi_wl_surface_get_present_modes` returns
      - `VK_PRESENT_MODE_MAILBOX_KHR`
      - `VK_PRESENT_MODE_FIFO_KHR`
    - `x11_surface_get_present_modes` returns
      - `VK_PRESENT_MODE_IMMEDIATE_KHR`
      - `VK_PRESENT_MODE_MAILBOX_KHR`
      - `VK_PRESENT_MODE_FIFO_KHR`
      - `VK_PRESENT_MODE_FIFO_RELAXED_KHR`
- `VK_KHR_get_surface_capabilities2`
  - `vkGetPhysicalDeviceSurfaceCapabilities2KHR`
    - `wsi_GetPhysicalDeviceSurfaceCapabilities2KHR`
  - `vkGetPhysicalDeviceSurfaceFormats2KHR`
    - `wsi_GetPhysicalDeviceSurfaceFormats2KHR`
- `VK_KHR_swapchain`
  - `vkCreateSwapchainKHR`
    - `wsi_CreateSwapchainKHR`
    - `wsi_wl_surface_create_swapchain` calls `wsi_wl_image_init` to allocate
      swapchain images
    - `x11_surface_create_swapchain` calls `x11_image_init` to allocate
      swapchain images
  - `vkDestroySwapchainKHR`
    - `wsi_DestroySwapchainKHR`
    - `wsi_wl_swapchain_destroy` frees the images and the swapchain
    - `x11_swapchain_destroy` frees the images and the swapchain
  - `vkGetSwapchainImagesKHR`
    - `wsi_GetSwapchainImagesKHR`
    - `wsi_wl_swapchain_get_wsi_image`
    - `x11_get_wsi_image`
  - `vkAcquireNextImageKHR`
    - `wsi_AcquireNextImageKHR`
    - `wsi_wl_swapchain_acquire_next_image` returns the next image that is not
      busy
    - `x11_acquire_next_image` returns the next image that is not busy
  - `vkQueuePresentKHR`
    - `wsi_QueuePresentKHR`
      - some drivers provide their entrypoints
    - `wsi_wl_swapchain_queue_present` calls `wl_surface_commit`
    - `x11_queue_present` calls `xcb_present_pixmap_checked`
  - `vkGetDeviceGroupPresentCapabilitiesKHR`
    - `wsi_GetDeviceGroupPresentCapabilitiesKHR`
  - `vkGetDeviceGroupSurfacePresentModesKHR`
    - `wsi_GetDeviceGroupSurfacePresentModesKHR`
  - `vkGetPhysicalDevicePresentRectanglesKHR`
    - `wsi_GetPhysicalDevicePresentRectanglesKHR`
    - `wsi_wl_surface_get_present_rectangles` returns `UINT32_MAX`
    - `x11_surface_get_present_rectangles` returns the current window geometry
  - `vkAcquireNextImage2KHR`
    - `wsi_AcquireNextImage2KHR`
- `VK_EXT_swapchain_maintenance1`
  - `vkReleaseSwapchainImagesEXT`
    - `wsi_ReleaseSwapchainImagesEXT`
    - `wsi_wl_swapchain_release_images` marks all images not busy
    - `x11_release_images` marks all images not busy
- `VK_KHR_present_wait`
  - `vkWaitForPresentKHR`
    - `wsi_WaitForPresentKHR`
    - `wsi_wl_swapchain_wait_for_present` uses `wp_presentation` protocol
    - `x11_wait_for_present` uses `XCB_PRESENT_EVENT_COMPLETE_NOTIFY`

## Swapchains

- overview
  - each VkImage is internally associated with a VkFence
  - when presenting, an empty `vkQueueSubmit` is called to wait the
    user-specified semaphores and signal the internal VkFence
    - this gives the driver a chance to set up implicit sync for the server
    - the submit is not empty but a blit when the display server uses a
      different GPU
  - the internal VkFence makes sure...
    - finer control over when to present to the server?
    - not presenting too fast?
- wayland
  - `wsi_wl_surface_create_swapchain`
    - call the common `wsi_swapchain_init` to intialize the swapchain
    - call `wsi_wl_display_init` to get the formats and modifiers
    - call `wsi_wl_image_init` for each image
      - create a `wsi_image` using `wsi_create_native_image`
      - create a `wl_buffer` using `zwp_linux_buffer_params_v1` or
      	`wl_drm_create_prime_buffer`
  - `wsi_wl_swapchain_destroy`
    - call `wl_buffer_destroy` and `wsi_destroy_image` for each image
    - call the common `wsi_swapchain_finish`
  - `wsi_wl_swapchain_get_wsi_image`
    - return the `wsi_image`
  - `wsi_wl_swapchain_acquire_next_image`
    - return one of the idle images and mark it busy
    - if all busy, wait for `wl_buffer_listener::release`
  - `wsi_wl_swapchain_queue_present`
    - mark the image busy and call `wl_surface_commit`
- x11
  - `x11_surface_create_swapchain`
    - call the common `wsi_swapchain_init` to intialize the swapchain
    - call `xcb_dri3_open` to check if same device or not; if not, will blit
      to a linear image on present
    - call `xcb_dri3_get_supported_modifiers` to get the modifiers, if
      supported
    - call `x11_image_init` for each image
      - create a `wsi_image` using `wsi_create_native_image` or
      	`wsi_create_prime_image`
      - create an `xcb_pixmap_t` using `xcb_dri3_pixmap_from_buffers_checked`
      	or `xcb_dri3_pixmap_from_buffer_checked` from the `wsi_image`
      - create a `xshmfence` on a shm
      - create a xsync fence using `xcb_dri3_fence_from_fd` from the
      	`xshmfence`
  - `x11_swapchain_destroy`
    - call `x11_image_finish` for each image
    - call the common `wsi_swapchain_finish`
  - `x11_get_wsi_image`
    - return the `wsi_image`
  - `x11_acquire_next_image`
    - return one of the idle images
    - if all busy, call `x11_handle_dri3_present_event` to handle x present
      events
      - when a present is handled by copying in the server, the server
      	triggers the fence and sends the idle notify immediately
      	- the GPU copying might still be going on!
      - whe a present is handled by flipping in the server, the server
      	triggers the fence and sends the idle notify for the prior pixmap (I
      	guess)
  - `x11_queue_present`
    - mark the image busy and call `xcb_present_pixmap`
- helpers
  - `wsi_swapchain_init`
    - call `vkCreateCommandPool` to create a command pool for each possible
      queue family
  - `wsi_swapchain_finish`
    - destroy per-queue-family `VkCommandPool`
    - destroy per-image `VkFence`
  - `wsi_create_native_image`
    - call `vkCreateImage` to createa a `VkImage`
    - call `vkAllocateMemory` to create and bind a `VkDeviceMemory`
    - call `vkGetMemoryFdKHR` to get the dmabuf
    - call `vkGetImageDrmFormatModifierPropertiesEXT` to get the modifier
    - call `vkGetImageSubresourceLayout` to get the layout
      - this is out-of-spec
  - `wsi_create_prime_image`
    - call `vkCreateBuffer` to createa a `VkBuffer`
    - call `vkAllocateMemory` to create and bind a `VkDeviceMemory` to the
      buffer
    - call `vkCreateImage` to createa a `VkImage`
    - call `vkAllocateMemory` to create and bind a `VkDeviceMemory` to the
      image
    - call `vkGetMemoryFdKHR` to get the dmabuf of the buffer memory
    - call `vkAllocateCommandBuffers` for each queue family
      - and set up a `vkCmdCopyImageToBuffer` from the image to the buffer
  - `wsi_common_queue_present` always throttles present by waiting on the
    fence associated with the previous present of the same image
    - in immediate mode, apps might acquires/presents crazily igoring what the
      spec says about throttling
    - if presenting is just blitting with GPU, the server may return the
      presented pixmap immediately
    - apps can be many frames (unbounded actually) ahead of what is presented

## X11 swapchain queue thread

- Present extension supports
  - present a pixmap at a `target_msc`, a specific vsync
  - if `XCB_PRESENT_OPTION_ASYNC` is set, and `target_msc` is in the past,
    present immediately
  - can have at most 1 in-flight non-async present at each `target_msc`
    - a second non-async present at the same `target_msc` replaces the first
      one, like mailbox mode
  - `XCB_PRESENT_EVENT_COMPLETE_NOTIFY` indicates that a present has completed
    (pixmap becomes on-screen)
  - `XCB_PRESENT_EVENT_IDLE_NOTIFY` indicates that a pixmap is no longer
    accessed by the server
- X11 WSI supports
  - `VK_PRESENT_MODE_IMMEDIATE_KHR`
    - no acquire/present queue (unless xwayland)
    - `x11_acquire_next_image` picks the first idle image
      - if all images are busy, it blocks for `XCB_PRESENT_EVENT_IDLE_NOTIFY`
    - `x11_queue_present` presents to server immediately
      - `target_msc` is set to 0
      - `XCB_PRESENT_OPTION_ASYNC` is set
  - `VK_PRESENT_MODE_MAILBOX_KHR`
    - a present queue is needed, such that kernel driver page flip is not
      blocked by implicit fence
    - `x11_acquire_next_image` picks the first idle image
      - if all images are busy, it blocks for `XCB_PRESENT_EVENT_IDLE_NOTIFY`
    - `x11_queue_present` adds the image to the present queue
    - the wsi thread pulls the first image in the present queue, wait until
      the associated fence is idle, and present
      - `target_msc` is set to 0
      - `XCB_PRESENT_OPTION_ASYNC` is not set (unless xwayland)
      - this gives mailbox mode
      - the fence wait makes sure the image can be flipped without being
      	blocked by implicit fencing in the kernel driver
  - `VK_PRESENT_MODE_FIFO_KHR` and `VK_PRESENT_MODE_FIFO_RELAXED_KHR`
    - require both acqure/present queues
    - `x11_acquire_next_image` pulls from the acquire queue
      - if empty, it blocks until the wsi thread pushes a new image to the
      	acquire queue after `XCB_PRESENT_EVENT_IDLE_NOTIFY`
    - `x11_queue_present` adds the image to the present queue
    - the wsi thread pulls the first image in the present queue and presents
      - `target_msc` is set to next vsync
      - `XCB_PRESENT_OPTION_ASYNC` is set only when relaxed
      - it then blocks waiting for `XCB_PRESENT_EVENT_COMPLETE_NOTIFY`
        - i.e., until next vsync, or later if implicit fencing gets in the way
- under xwayland, there are a few changes
  - there are two limits of wayland
    - wayland protocol does not support async present
    - wayland compositors usually compose without checking the implicit fence,
      which can lead to missed vsyncs just because an app's buffer is
      GPU-heavy; some compositors check and latch an app's buffer only when it
      is ready
  - due to the limits, the first change is that mailbox presents set
    `XCB_PRESENT_OPTION_ASYNC`
    - the wsi thread waits for fences before present
    - this gets the buffer from xwayland to the wayland compositor asap
    - the wayland protocol ignores async
  - the second change is that immediate present uses the present queue and
    waits for fences first
    - this is because wayland compositors usually compose without checking the
      implicit fence, causing missed vsyncs
- in fifo mode, both an acquire queue and an present queue are used
  - `vkAcquireNextImageKHR` calls `wsi_queue_pull` on the acquire queue
  - `vkQueuePresentKHR` calls `wsi_queue_push` on the present queue
  - life cycle of an image
    - `wsi_queue_pull` waits until `XCB_PRESENT_EVENT_IDLE_NOTIFY`.  The event
      clears image `busy`, decrements `sent_image_count`, and adds the image
      to the acquire queue for pulling.
    - app renders to the image
    - `wsi_queue_push` adds the image to the present queue
    - the present queue thread
      - pulls the image from the present queue
      - calls `x11_present_to_x11_dri3` to increment `sent_image_count`, sets
      	image `present_queued`, and calls `xcb_present_pixmap`
      - waits until `XCB_PRESENT_EVENT_COMPLETE_NOTIFY` which clears
      	`present_queued`
  - from perfetto
    - app thread is blocked in `vkAcquireNextImageKHR`
    - `wl_surface::frame` callback wakes up sommelier
    - sommelier wakes up Xwayland
    - Xwayland wakes up present queue thread with
      `XCB_PRESENT_EVENT_IDLE_NOTIFY` followed by
      `XCB_PRESENT_EVENT_COMPLETE_NOTIFY`
    - present queue thread handles the idle event and wakes up app thread with
      `wsi_queue_push`
      - app thread renders and `vkQueuePresentKHR` to add the next frame to
      	the present queue
      - app thread calls `vkAcquireNextImageKHR` and is blocked
    - present queue thread handles the complete event and exits the wait loop
    - present queue thread pulls the last frame from the present queue and
      wakes up Xwayland with `xcb_present_pixmap`
      - Xwayland wakes up sommelier
      - sommelier sends `wl_surface::attach`, `wl_surface::frame`,
        `wl_surface::damage`, and `wl_surface::commit`
      - `wl_buffer::release` wakes up sommelier
      - sommelier wakes up X
      - X???
    - present queue thread blocks in `xcb_wait_for_special_event` again
- in immediate mode on Xwayland, only the present queue is used
  - immediate mode on xwayland is treated as mailbox mode
  - app prepares for the next frame (doing all works not depending on the
    image of the next frame)
  - when app is ready to render, it calls `vkAcquireNextImageKHR`
  - app renders to the acquired image and calls `vkQueuePresentKHR`
  - present queue thread is woken up, wait for rendering to complete, handle
    idle and complete events (from last frame), and presents the frame to X

## Drivers

- `wsi_device_init`
  - wsi uses `VK_EXT_pci_bus_info` to get the pci bus info
    - used to check if server and driver uses the same drm device
    - x11 only, missing on wayland
- `wsi_device` knobs
  - `supports_modifiers` sets whether wsi should use modifiers or not
  - `signal_semaphore_for_memory` and `signal_fence_for_memory` is called on
    newly acquired image
    - the semaphore/fence will be signaled when the acquired image is ready
    - used by anv such that they are signaled only when the bo is idle (i.e.,
      display engine retires the bo)
  - `set_memory_ownership` sets ownership to true on newly acquired image and
    to false after queue present call
    - used by radv; `VK_EXT_descriptor_indexing` or
      `VK_KHR_buffer_device_address` requires radv to use global bo list.  It
      needs to add the wsi bo to the list after the bo is acquired but before
      present
- driver requirements
  - `vkGetMemoryFdKHR` must work
    - to share dma-buf to the server
  - `vkGetImageSubresourceLayout` must work for tiled images
    - offset and stride are needed to create `wl_buffer` or `xcb_pixmap_t`
- private extensions
  - `wsi_image_create_info`
    - chained to `VkImageCreateInfo` for swapchain images
    - specifies that the image is for scanout
    - used only when there is no modifier
      - anv uses it to force linear or X tiling
      - radv uses it to mark for scanout and disable DCC
      - turnip uses it to force `DRM_FORMAT_MOD_LINEAR`
  - `wsi_memory_allocate_info`
    - chained to `VkMemoryAllocateInfo` for swapchain images
    - specifies that the bo should enable implicit sync
    - used by radv to set `RADEON_FLAG_IMPLICIT_SYNC` for the bo
    - used by turnip to set `MSM_SUBMIT_NO_IMPLICIT` for all submits
  - `wsi_surface_supported_counters`
    - chained to `VkSurfaceCapabilities2KHR` for display surface
  - `wsi_memory_signal_submit_info`
    - chained to empty (or blit) `VKQueueSubmit` just before presentation
    - gives driver a chance to set up implicit sync for the bo
    - used by anv to add a `EXEC_OBJECT_WRITE` reloc for implicit sync

## Displays

- related extensions
  - `VK_KHR_display`
    - `VK_KHR_get_display_properties2`
    - `VK_KHR_display_swapchain`
  - `VK_EXT_direct_mode_display`
    - `VK_EXT_acquire_xlib_display`
    - `VK_EXT_acquire_drm_display`
    - `VK_EXT_display_surface_counter`
    - `VK_EXT_display_control`
- basic usage only needs `VK_KHR_display` and `VK_KHR_get_display_properties2`
  - `vkGetPhysicalDeviceDisplayPropertiesKHR2` enumerates `VkDisplayKHR`s
  - `vkGetDisplayModeProperties2KHR` enumerates `VkDisplayModeKHR`s of a
    `VkDisplayKHR`
  - `vkGetPhysicalDeviceDisplayPlaneProperties2KHR` returns hw plane count as
    well as `VkDisplayKHR`s to which hw planes are currently attached
  - `vkGetDisplayPlaneSupportedDisplaysKHR` returns `VkDisplayKHR`s with which
    a hw plane is compatible
  - `vkGetDisplayPlaneCapabilities2KHR` returns capabilities of a hw plane
  - `vkCreateDisplayPlaneSurfaceKHR` creates a `VkSurfaceKHR` from
    `VkDisplayModeKHR` and hw plane index
  - a swapchain can then be created using `VK_KHR_swapchain`
    - `VK_KHR_display_swapchain` is about swapchains that share images and is
      optional
- display leasing uses `VK_EXT_direct_mode_display` and platform-specific
  extensions
  - `vkAcquireXlibDisplayEXT` leases an X11 display using
    `xcb_randr_create_lease` for direct access
  - `wsi_ReleaseDisplayEXT` releases the X11 display back to X11
- internals
  - when `VK_KHR_display` is enabled, `vkCreateDevice` opens the render node
    as well as the primary node
  - `wsi_device_init` calls `wsi_display_init_wsi` to initialize `wsi_display`
  - `wsi_get_connectors` calls `drmModeGetResources` to discover all
    connectors and all modes
    - for each drm connector, a `wsi_display_connector` is created
    - for each drm mode of each connector, a `wsi_display_mode` is created
    - a `VkDisplayKHR` is a `wsi_display_connector`
    - a `VkDisplayModeKHR` is a `wsi_display_mode`
  - wsi assumes each drm connector has a hw plane
    - that sounds bad
    - as such, `wsi_GetDisplayPlaneSupportedDisplaysKHR` always returns 1
      `VkDisplayKHR`
    - `wsi_GetDisplayPlaneCapabilities2KHR` just lies

## Implicit Fencing

- mesa assumes compositors require implicit fencing
- on modern kernels, wsi uses dma-buf sync file import/export to bridge vk
  drivers (explicit fencing) and compositors (implicit fencing)
  - on present, `wsi_common_queue_present` makes a empty or blit queue submit
    - `wsi_prepare_signal_dma_buf_from_semaphore` creates
      `chain->dma_buf_semaphore` once
    - the empty or blit queue submit signals `chain->dma_buf_semaphore`
    - `wsi_signal_dma_buf_from_semaphore` exports the sync fd from
      `chain->dma_buf_semaphore` and imports the sync fd into
      `image->dma_buf_fd`
  - on acquire, `wsi_common_acquire_next_image2` calls
    `wsi_signal_semaphore_for_image` and `wsi_signal_fence_for_image` to
    signal the `VkSemaphore` and the `VkFence`
    - they call to `wsi_create_sync_for_dma_buf_wait` exports the sync fd from
      `image->dma_buf_fd`, wraps it in a `vk_sync`, and update
      `semaphore->temporary` and `fence->temporary`
- on older kernels, wsi uses vk driver callbacks to set up implicit fencing
  - on present, `wsi_common_queue_present` chains
    `wsi_memory_signal_submit_info` into the empty or blit submit
    - this is only used by anv and allows anv to add an implicit fence to the
      bo
    - other drivers marks the bo for implicit fencing on bo allocation
  - on acquire, `wsi_common_acquire_next_image2` can call
    `dev->create_sync_for_memory` to create a `vk_sync` for the bo's implicit
    fence set up by the compositor
    - this is only used by anv
    - other drivers marks the bo for implicit fencing on bo allocation
