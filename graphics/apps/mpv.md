mpv
===

## buid

- steps
  - `git clone https://github.com/mpv-player/mpv.git`
  - `apt install libplacebo-dev libass-dev`
  - `meson setup out`
  - `ninja -C out`
  - `./out/mpv`

## usage

- `mpv -v <file>` shows that
  - `demux` uses `libavformat`
  - `vo` uses `vo/gpu/opengl` and `vo/gpu/x11`
  - `vd` uses sw
- `--hwdec=vaapi` to use vaapi for `vd`
  - or `--hwdec=auto` to try all supported apis

## case study

- `mpv --no-config --hwdec=vaapi foo.mkv`
- stack
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
