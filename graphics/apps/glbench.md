GLBench
=======

## Overview

- it is really outdated
- for each test,
  - `GLInterface::Create`
  - `GLInterface::Init`
  - `ClearBuffers`
    - this calls `SwapBuffers` twice
  - `TestBase::Run`
  - `GLInterface::Cleanup`
- `TestBase::Run`
  - set up resources for the test
  - `RunTest`
    - `Bench` waits for machine to cool down and calls `TimeTest`
    - `TimeTest` times
      - `GLInterface::SwapBuffers`
      - `glFinish`
      - `TestBase::TestFunc`
      - `glFinish`

## `WaffleInterface`

- on cros, glbench uses an unofficial outdated "null" version of waffle plus
  cros's own patches
- `WaffleInterface::InitOnce`
  - `waffle_init` with `WAFLE_PLATFORM_NULL`
  - `waffle_display_connect`
  - `waffle_config_choose`
  - `waffle_window_create`
  - `waffle_window_show`
- `WaffleInterface::Init`
  - `waffle_context_create`
  - `waffle_make_current`
  - `waffle_get_proc_address`
- `WaffleInterface::SwapBuffers`
  - `waffle_window_swap_buffers`
- `WaffleInterface::Cleanup`
  - `waffle_make_current` with `NULL`
  - `waffle_context_destroy`
  - it does not call `waffle_window_destroy`, `waffle_display_disconnect`, nor
    `waffle_teardown`

## Waffle NULL platform

- outdated
- it uses EGL on GBM, which is also known as "surfaceless"
- `wnull_platform_create` calls `wgbm_platform_init` and then override
  `EGL_PLATFORM` and vtbl
- `wnull_display_connect` scans `/dev/dri/card%d` and uses the first node that
  has connectors
  - it creates a `gbm_device` over the fd
  - it picks the preferred `drmModeConnectorPtr`, `drmModeModeInfoPtr`, and
    `drmModeCrtcPtr`
  - it calls `wegl_display_init` to initialize EGL with `EGL_DEFAULT_DISPLAY`
- `wnull_window_create`
  - `vsync_wait` is true by default
  - `show_method` is `WAFFLE_WINDOW_NULL_SHOW_METHOD_COPY_GL` by default
- `wnull_make_current`
  - it calls `eglMakeCurrent` without surfaces (`GL_OES_surfaceless_context`)
  - it calls `wnull_window_prepare_draw_buffer`
    - `slbuf_bind_fb` calls `slbuf_get_glfb` which create a gl fbo
      - it uses `gbm_bo_create` to create a bo
      - it uses `eglCreateImageKHR` to import the dmabuf as `EGLImage`
      - it uses `glEGLImageTargetRenderbufferStorageOES` to bind the image
- `wnull_window_swap_buffers`
  - `wnull_display_present_buffer` creates a scanout `gbm_bo` and copies from
    gl fbo to the scanout bo, because the method is
    `WAFFLE_WINDOW_NULL_SHOW_METHOD_COPY_GL`
  - it calls `glFinish` twice to wait for rendering and then wait for copying
  - it calls `drmModeAddFB` for the scanout bo
  - it calls `drmModeSetCrtc` or `drmModePageFlip` to present
