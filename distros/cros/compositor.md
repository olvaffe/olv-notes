Chrome OS Compositor
====================

## Overview

- Front buffers
  - create "gbm_bo"
  - create drm fb using "drmModeAddFB2" from "gbm_bo" drm handle
  - create "EGLImage" from "gbm_bo" dmabufs
  - create gl fbos around "EGLImage"
- Swap chain
  - "drmModeSetCrtc"
  - GL draw and "glFinish"
  - "drmModePageFlip"

## GBM, Generic Buffer Manager

- "gbm_device" is a wrapper around DRM fd
  - usage limited to scanout, cursor, rendering, (CPU) write, and linear (tiling)
  - "is_format_supported", but usage is mostly ignored
  - "get_format_modifier_plane_count", and usage is omitted
  - "bo_create" with width/height/format/usage
  - "bo_create_with_modifiers" with modifiers but no usage
  - "bo_import" with usage
     - support WL_BUFFER, EGL_IMAGE, FD, and FD_PLANAR
     - buffer width/height/stride/format needs to be specified
  - "surface_create" with width/height/format/usage
  - "surface_create_with_modifiers" with modifiers but no usage
- "gbm_bo" is a buffer object
  - allocated or imported
  - point back to the "gbm_device"
  - width, height, format, modifier, bpp, stride, planar {strides,offsets} can be queried
  - "get_handle" to return platform-specific handle
  - "get_fd" to create a DMA-BUF (aka PRIME) fd, owned by the caller
  - "set_user_data" to assign a user data
  - "write" to write data into the BO
  - "map" and "unmap" for subregion mmaped access (might use a staging buffer)
- "gbm_surface" is a swap chain
  - it turns GBM into an EGL platform
  - there are initially N (>=1) free buffers
  - "has_free_buffers" must be called to make sure there are still free buffers
  - any EGL/GLES call probably requires and dequeues a free buffer
  - "eglSwapBuffers" queues the buffer back
  - "lock_front_buffer" acquires the queued buffer
  - "release_buffer" releases the acquired buffer
    - when N > 1, no need to release acquired buffer immediately each frame
- "gbm_dri.so" implementation
  - "gbm_device" internally consists of "__DRIscreen" and "__DRIcontext"
  - "gbm_bo" internally consists of "__DRIimage"
  - "gbm_surface" internally is implemented by EGL
    - that is why it is understood and can be used with EGL
- "minigbm" implementation
  - no "gbm_surface" support
  - only FD and FD-PLANAR imports
  - many more BO usage flags
  - some more BO queries (plane size, tiling, etc.)

## DRM/KMS

- https://dri.freedesktop.org/docs/drm/gpu/index.html
- echo 0x3f > /sys/module/drm/parameters/debug
- How to program DRM/KMS
  - drmModeGetResources
    - drmModeGetConnector
    - drmModeGetEncoder
    - to determine crtc
  - drmModeAddFB2 or drmModeAddFB2WithModifiers to add fb
    - for each potential scanout buffers
  - drmModeSetCrtc to set the crtc/fb/connector/mode
  - drmModePageFlip to flip fb
  - drmHandleEvent to wait for flip to finish

## EGL/GLES

- Platform extensions
  - EGL_MESA_platform_gbm
    - EGLNativeDisplayType is gbm_device
      - EGL_DEFAULT_DISPLAY works as well
    - EGLNativeWindowType is gbm_surface
  - EGL_MESA_platform_surfaceless
    - EGLNativeDisplayType must be EGL_DEFAULT_DISPLAY
    - No EGLNativeWindowType nor window surfaces
    - eglSwapBuffers only has effect on window surfaces by definition; it has
      no effect on this platform
- Context extensions
  - EGL_KHR_surfaceless_context
    - eglMakeCurrent without any surface
  - EGL_KHR_no_config_context
    - eglCreateContext without any EGLConfig
- Image extensions
  - to import DMA-BUFs as images for both sampling and rendering
  - EGL_KHR_image_base
  - EGL_EXT_image_dma_buf_import
  - EGL_EXT_image_dma_buf_import_modifiers
  - GL_OES_EGL_image
  - GL_OES_EGL_image_external
- Fence extensions
  - EGL_KHR_fence_sync
    - insert a sync object into the command stream
    - client-side waiting
  - EGL_KHR_wait_sync
    - server-side waiting
- drm-tests requires
  - EGL_KHR_surfaceless_context
  - DMA-BUF import as FBO

## Vulkan

- Memory extensions
  - allow VkDeviceMemory <-> dma_buf
  - VK_KHR_external_memory_capabilities
  - VK_KHR_external_memory
  - VK_KHR_external_memory_fd
  - VK_EXT_external_memory_dma_buf
- Sync extensions
  - VK_KHR_external_semaphore
  - VK_KHR_external_fence
- VK_EXT_image_drm_format_modifier
- VK_EXT_queue_family_foreign
- VK_ANDROID_external_memory_android_hardware_buffer
- https://github.com/SaschaWillems/Vulkan

## Tests

- drm-tests
  - https://chromium.googlesource.com/chromiumos/platform/drm-tests/

## Wayland

- XDG_RUNTIME_DIR=/var/run/chrome
