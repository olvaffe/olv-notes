Mesa Vulkan WSI
===============

## Overview

- it provides 1:1 helpers for most WSI entrypoints
- `VK_KHR_wayland_surface`
  - `vkCreateWaylandSurfaceKHR`
    - `wsi_create_wl_surface`
    - allocates and initializes `VkIcdSurfaceWayland` as required by the
      loader
  - `vkGetPhysicalDeviceWaylandPresentationSupportKHR`
    - `wsi_wl_get_presentation_support`
    - checks if `wl_drm` (old) or `zwp_linux_dmabuf_v1` (new) is working
- `VK_KHR_xcb_surface`
  - `vkCreateXcbSurfaceKHR`
    - `wsi_create_xcb_surface`
    - allocates and initializes `VkIcdSurfaceXcb` as required by the loader
  - `vkGetPhysicalDeviceXcbPresentationSupportKHR`
    - `wsi_get_physical_device_xcb_presentation_support`
    - checks for DRI3 and if the screen depth is 24/32 bits
- `VK_KHR_surface`
  - `vkDestroySurfaceKHR`
    - no helper; just free the struct
  - `vkGetPhysicalDeviceSurfaceSupportKHR`
    - `wsi_common_get_surface_support`
    - for wayland, always true
    - for xcb, checks for DRI3 and if the window depth is 24/32 bits
  - `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`
    - `wsi_common_get_surface_capabilities`
    - for wayland,
      - current/min/max extents are set `UINT32_MAX`/1x1/gpu max extent
      - `minImageCount` is 4
    - for xcb
      - current/min/max extents are set to the current window geometry
      - `minImageCount` is 3
    - in both cases, the images support these usage flags
      - transfer src/dst
      - sampled
      - storage
      - color attachment
  - `vkGetPhysicalDeviceSurfaceFormatsKHR`
    - `wsi_common_get_surface_formats`
    - for wayland, return formats advertised by `wl_drm` or
      `zwp_linux_dmabuf_v1`
    - for x11, return `VK_FORMAT_B8G8R8A8_UNORM` and `VK_FORMAT_B8G8R8A8_SRGB`
  - `vkGetPhysicalDeviceSurfacePresentModesKHR`
    - `wsi_common_get_surface_present_modes`
    - for wayland, return mailbox and fifo
    - for x11, return all present modes: immediate, mailbox, fifo, fifo
      relaxed
- `VK_KHR_get_surface_capabilities2`
  - `vkGetPhysicalDeviceSurfaceCapabilities2KHR`
    - `wsi_common_get_surface_capabilities2`
  - `vkGetPhysicalDeviceSurfaceFormats2KHR`
    - `wsi_common_get_surface_formats2`
- `VK_KHR_swapchain`
  - `vkCreateSwapchainKHR`
    - `wsi_common_create_swapchain`
  - `vkDestroySwapchainKHR`
    - `wsi_common_destroy_swapchain`
  - `vkGetSwapchainImagesKHR`
    - `wsi_common_get_images`
  - `vkAcquireNextImageKHR`
  - `vkQueuePresentKHR`
    - `wsi_common_queue_present`
  - `vkGetDeviceGroupPresentCapabilitiesKHR`
  - `vkGetDeviceGroupSurfacePresentModesKHR`
  - `vkGetPhysicalDevicePresentRectanglesKHR`
    - `wsi_common_get_present_rectangles`
    - for wayland, return `UINT32_MAX`
    - for x11, return the current window geometry
  - `vkAcquireNextImage2KHR`
    - `wsi_common_acquire_next_image2`

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
  - `wsi_memory_allocate_info`
    - chained to `VkMemoryAllocateInfo` for swapchain images
    - specifies that the bo should enable implicit sync
    - used by radv to set `RADEON_FLAG_IMPLICIT_SYNC`
  - `wsi_memory_signal_submit_info`
    - chained to empty (or blit) `VKQueueSubmit` just before presentation
    - gives driver a chance to set up implicit sync for the bo
    - used by anv to add a `EXEC_OBJECT_WRITE` reloc for implicit sync
