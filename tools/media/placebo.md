libplacebo
==========

## Overview

- <https://code.videolan.org/videolan/libplacebo.git>
  - `git clone --recursive https://code.videolan.org/videolan/libplacebo.git`
  - `meson setup out -Dbuildtype=debug`
- rewrite of mpv core rendering algorithms

## API

- high-level renderer
  - `options.h` parses options from string form to params
  - `renderer.h` is a rendering pipeline that converts an input image to a
    desired output image
  - `utils/frame_queue.h` abstracts a stream of input images (e.g., a video
    decoder)
  - `utils/upload.h` uploads pixel data to textures
  - `utils/dav1d.h` converts a dav1d picture to a `pl_frame`
  - `utils/libav.h` converts a `libav*` object to a placebo object
- shader dispatch
  - `dispatch.h` generates complete shaders from stubs, and dispatches them to
    gpu for execution
- shader generation
  - `shaders.h` generates shader stubs for various algorithms
- gpu abstraction
  - `gpu.h` is an abstraction of the gpu
  - `swapchain.h` is an abstraction of the swapchain
  - `vulkan.h` exposes the underlying vulkan instance/device
  - `opengl.h` exposes the underlying opengl context
- utils
  - `cache.h` is the cache subsystem for large blobs such as compiled shaders,
    3D luts, etc.
  - `colorspace.h` is color space definitions and conversions on cpu
  - `log.h` is logging
  - `dither.h` generates noise and dither matrices on cpu
  - `filters.h` generates filter kernels on cpu
  - `tone_mapping.h` tone-maps between SDR/HDR on cpu
  - `gamut_mapping.h` gamut-maps standard gamut and wide gamut on cpu

## Demos

- headless vulkan
  - `video-filtering`
    - `init`
      - `pl_vulkan_create` to create `pl_vulkan`
      - `pl_dispatch_create` to create `pl_dispatch`
    - `api1_reconfig`
      - `pl_tex_recreate` to create `pl_tex`
    - `api1_filter`
      - `pl_tex_upload` to upload `pl_tex`
      - `pl_dispatch_finish` to dispatch the job
      - `pl_tex_download` to download `pl_tex`
  - `multigpu-bench`
- supported winsys/api
  - glfw and sdl
  - vulkan and opengl
  - `sdl_create`
    - `SDL_Init(SDL_INIT_VIDEO)` to initialize sdl
    - `SDL_CreateWindow` to create a window
    - `pl_vk_inst_create` to create a `pl_vk_inst`
    - `SDL_Vulkan_CreateSurface` to create a surface
    - `pl_vulkan_create` to create a `pl_vulkan`
    - `pl_vulkan_create_swapchain` to create a `pl_swapchain`
- `colors`
  - `window_create` creates a window
  - `pl_swapchain_start_frame` gets a `pl_swapchain_frame`
  - `pl_tex_clear` clears the frame
  - `pl_swapchain_submit_frame` submits a frame for presentation
  - `pl_swapchain_swap_buffers` waits for some previously submitted frame to
    be ready
    - this is for throttling and the exact definition is vague
- `sdlimage` depends on `SDL2_image`
  - `IMG_Load` loads an image
  - `window_create` creates a window
  - `pl_upload_plane` uploads the image to `pl_plane` and `pl_tex`
  - `pl_renderer_create` creates a renderer
  - the main loop is roughly the same as `colors`, except `render_frame` is
    used to render a frame
    - `pl_frame_from_swapchain` gets a `pl_frame` from a `pl_swapchain_frame`
    - `pl_render_image` renders the image to the frame
- `plplay` depends on ffmpeg
  - `open_file` loads a video
  - `window_create` creates a window
  - `pl_test_pixfmt` tests for gpu support for the video pixfmt
  - `init_codec` inits the codec, with optional hwdec support
  - `pl_queue_create` creates a `pl_queue`
  - `pl_thread_create` creates a `pl_thread` for `decode_loop`
    - `av_read_frame` reads a packet
    - `avcodec_send_packet` sends the packet for decoding
    - `avcodec_receive_frame` reads a decoded frame
    - `pl_queue_push_block` adds the decoded frame to the queue
  - `pl_renderer_create` creates a renderer
  - `render_loop` is the main loop
    - `pl_swapchain_start_frame` acquires the swapchain frame
    - `pl_queue_update` queries the queue and returns a `pl_frame_mix`
      - `map_entry` calls `map_frame` to map `pl_source_frame` to `pl_frame`
      - the source frame is an `AVFrame`
      - `pl_map_avframe_ex` does the work; when hwdec,
        - `av_hwframe_map` maps `AV_PIX_FMT_VAAPI` to `AV_PIX_FMT_DRM_PRIME`
          - this calls `vaapi_map_from` which calls `vaExportSurfaceHandle`
          - because `AV_HWFRAME_MAP_READ` is set, this implies `vaSyncSurface`
        - `pl_map_avframe_drm` maps `AV_PIX_FMT_DRM_PRIME` to `pl_tex`
    - `render_frame` renders a frame
      - `pl_render_image_mix` renders the `pl_frame_mix`
    - `pl_swapchain_submit_frame` submits the frame
    - `pl_swapchain_swap_buffers` throttles
