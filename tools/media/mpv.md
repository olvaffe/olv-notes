mpv
===

## build

- `git clone https://github.com/mpv-player/mpv.git`
- `apt install libplacebo-dev libass-dev`
  - or `pacman -S ffmpeg`
- `meson setup out -Dbuildtype=debug`
- `ninja -C out`
- `./out/mpv`

## usage

- `mpv -v <file>` shows that
  - `demux` uses `libavformat`
  - `vd` uses sw
  - `vf` always has `userdeint`, `autorotate`, and `convert`
    - they are nop if not needed
  - `vo` uses `vo/gpu/opengl` and `vo/gpu/x11`
    - or `vo/gpu/wayland` on wayland
- `--demuxer=help` shows the available demuxers
  - `lavf` is the default and uses `libavformat`
- `--vd=help` shows the available video codecs
  - they are all from `libavcodec`
    - a subset of `ffmpeg -decoders`
  - mpv picks the best codec automatically
  - `--hwdec` to enable hw
    - `--hwdec=no` is the default and uses sw
    - `--hwdec=auto-safe` or `--hwdec=yes` enables all allowlisted codecs
    - `--hwdec=auto` enables all codecs
    - `--hwdec=<codec>` enables the specified codec
      - a subset of `ffmpeg -hwaccels`
      - the interesting ones are `vaapi` and `vulkan`
    - `mpv -v --hwdec=vaapi <file>` shows that
      - vaapi/opengl interop uses `GL_EXT_EGL_image_storage`
      - sw often uses `yuv420p` and vaapi often uses `vaapi[nv12]`
- `--vf=help` shows the available video filters
  - there are always 3 internal filters that cannot be enabled/disabled
    - `userdeint`, `autorotate`, and `convert`
- `--vo=help` shows the available video outputs
  - `gpu` is the default
  - `--gpu-*` configs the `gpu` video output
    - `--gpu-debug` enables debug messages
    - `--gpu-context` specifies the GPU context
      - `--gpu-context=help` shows the available contexts
      - `x11egl` uses EGL and `EGL_KHR_platform_x11`
      - `wayland` uses EGL and `EGL_KHR_platform_wayland`
      - `drm` uses EGL and `EGL_KHR_platform_gbm`
      - `waylandvk` uses Vulkan and `VK_KHR_WAYLAND_SURFACE_EXTENSION_NAME`
    - `--gpu-api` specifies the GPU api
      - `--gpu-api=help` shows the available apis
      - `opengl` uses opengl
      - `vulkan` uses vulkan
      - mpv will auto-select the gpu context
  - `--opengl-*` configs the `gpu/opengl` video output
    - `--opengl-es` uses `EGL_OPENGL_ES2_BIT` rather than `EGL_OPENGL_BIT`
    - `--opengl-swapinterval` sets `eglSwapInterval`, defaulting to 1 meaning
      vsync'ed
    - `--opengl-waitvsync` calls the legacy `glXWaitVideoSyncSGI`
    - `--opengl-pbo` uses PBOs to upload decoded frames as textures
    - `--opengl-glfinish` calls `glFinish` after rendering and before buffer
      swapping
    - `--opengl-early-flush` calls `glFlush` after rendering and before buffer
      swapping
  - `--vulkan-*` configs the `gpu/vulkan` video output
    - `--vulkan-device` selects the physical device
    - `--vulkan-swap-mode` selects `VkPresentModeKHR`
    - `--vulkan-queue-count` limits the number of `VkQueue` to use
    - `--vulkan-async-compute` enables a separate compute-only queue for async
      copy and blit
    - `--vulkan-async-transfer` eanbles a separate transfer-only queue for
      async compute

## case study

- `mpv --no-config --hwdec=vaapi foo.mkv`
- main thread stack
  - `main`
  - `mpv_main`
  - `mp_play_files`
  - `play_current_file`
  - `run_playloop`
  - `write_video`
  - `video_output_image`
    - `mp_pin_out_read`
      - `mp_pin_out_request_data`
      - `filter_recursive`
      - `mp_filter_graph_run`
      - `vd_lavc_process`
      - `lavc_process`
        - `send_packet`
          - `avcodec_send_packet`
        - `receive_frame`
          - `decode_frame`
          - `avcodec_receive_frame`
    - `add_new_frame`
  - `vo_queue_frame` wakes up the vo thread
- vo thread stack
  - `vo_thread`
  - `render_frame`
  - `draw_frame` from `vo_gpu.c`
    - `gl_video_render_frame`
    - `pass_render_frame`
      - `pass_upload_image`
        - `ra_hwdec_mapper_map`
        - `mapper_map` from `hwdec_vaapi.c`
          - `vaExportSurfaceHandle`
          - `vaSyncSurface`
          - `vaapi_gl_map` calls `eglCreateImageKHR` and
            `glEGLImageTargetTexture2DOES`
  - `flip_page` from `vo_gpu.c`
    - `ra_gl_ctx_swap_buffers`
    - `wayland_egl_swap_buffers` calls `eglSwapBuffers`
