Piglit
======

## Build

- build
  - `git clone https://gitlab.freedesktop.org/mesa/piglit.git`
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug`
- build options
  - `PIGLIT_BUILD_GL_TESTS`
    - default ON
  - `PIGLIT_BUILD_GLES1_TESTS`
    - default ON on linux
    - requires waffle and egl
  - `PIGLIT_BUILD_GLES2_TESTS`
    - default ON on linux
    - requires waffle and egl
  - `PIGLIT_BUILD_GLES3_TESTS`
    - default ON on linux
    - requires waffle and egl
  - `PIGLIT_BUILD_CL_TESTS`
    - default OFF
    - requires opencl
  - `PIGLIT_BUILD_VK_TESTS`
    - default ON on linux
    - requires vulkan and glslang
  - `PIGLIT_BUILD_GLX_TESTS`
    - default ON if x11 and glx are found
  - `PIGLIT_BUILD_DMA_BUF_TESTS`
    - default ON if drm, gbm, and xcb-dri2 are found
- build system
  - the top-level `CMakeLists.txt` has `add_subdirectory(cmake/target_api)`
  - `cmake/target_api/CMakeLists.txt` has
    - `add_subdirectory(no_api)`
    - `add_subdirectory(gl)`
    - `add_subdirectory(gles1)`
    - `add_subdirectory(gles2)`
    - `add_subdirectory(gles3)`
    - `add_subdirectory(cl)`
    - they build `tests` with different `piglit_target_api`
  - `CMakeLists.txt` under `tests` usually has `piglit_include_target_api()`
    - the function is defined in `cmake/piglit_util.cmake`
    - it includes `CMakeLists.${piglit_target_api}.txt`
- run
  - `PIGLIT_BUILD_DIR=out ./piglit run -p surfaceless_egl -t 'dma_buf' quick_gl results`
  - this calls `run` from `framework/programs/run.py`
  - `load_test_profile` imports `tests/quick_gl.py`

## Tests

- use `spec/ext_image_dma_buf_import` as an example
- `CMakeLists.txt`
  - `CMakeLists.gles1.txt` and `CMakeLists.gles1.txt` build different subsets
    of available tests
  - they link against `piglitutil_gles1` and `piglitutil_gles2` respectively
- all tests have
  - `PIGLIT_GL_TEST_CONFIG_BEGIN` which expands to
    - `piglit_general_init` is nop on linux
    - `piglit_gl_test_config_init` zeros out `piglit_gl_test_config` and sets
      `config->window_width`/`config->window_height` to default values
    - `config.init = piglit_init` where `piglit_init` is defined by the test
    - `config.display = piglit_display` where `piglit_display` is defined by the
      test
  - `PIGLIT_GL_TEST_CONFIG_END` which expands to
    - `piglit_gl_process_args` parses args
      - `-auto` sets `piglit_automatic`
      - `-fbo` sets `piglit_use_fbo`
      - `-png` sets `piglit_dump_png`
      - more
    - `piglit_gl_test_run` runs the test according to `piglit_gl_test_config`
- `piglit_gl_test_run`
  - if waffle is available, it uses `piglit_fbo_framework_create` or
    `piglit_winsys_framework_factory` depending on `piglit_use_fbo`
    - `PIGLIT_PLATFORM` selects different waffle platforms
      - the default is `mixed_glx_egl`, which selects
        `WAFFLE_PLATFORM_X11_EGL` or `WAFFLE_PLATFORM_GLX`
      - `gbm` selects `WAFFLE_PLATFORM_GBM`
      - `glx` selects `WAFFLE_PLATFORM_GLX`
      - `x11_egl` selects `WAFFLE_PLATFORM_X11_EGL`
      - `wayland` selects `WAFFLE_PLATFORM_WAYLAND`
      - `surfaceless_egl` selects `WAFFLE_PLATFORM_SURFACELESS_EGL`
  - otherwise, it uses `piglit_glut_framework_create`
  - gl is initialized as a part of framework creation
  - `run_test` callback
    - when `-fbo` and `-auto`, it calls `config->init`, `config->display`, and
      `piglit_report_result` to report and exit
    - otherwise, it shows a window and enters the mainloop
      - the mainloop calls `config->display` to draw the window
      - if `-auto`, it calls `piglit_report_result` to report and exit

## `ext_image_dma_buf_import-sample_yuv`

- `piglit_init`
  - requires `EGL_EXT_image_dma_buf_import` and `GL_OES_EGL_image_external`
  - parses `-fmt=<fourcc>`
- `piglit_display`
  - calls `glClear` to clear color and depth
  - calls `piglit_create_dma_buf` to create a 4x4 dmabuf
  - `sample_buffer`
    - `egl_image_for_dma_buf_fd` creates an egl image for the dmabuf
    - `texture_for_egl_image` creates a gl texture for the egl image
    - `sample_tex` draws a quad
      - vs
        - `texcoords = piglit_texcoords.xy;`
        - `gl_Position = piglit_vertex;`
      - fs
        - `uniform samplerExternalOES sampler;`
        - `gl_FragColor = texture2D(sampler, texcoords);`
