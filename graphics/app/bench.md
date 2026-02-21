GPU Benchmarks
==============

## glmark2

- <https://github.com/glmark2/glmark2>
  - `./src/glmark2-es2-wayland --data-path ~/projects/glmark2/data -b build:model=bunny`
  - `-l` to see all scenes (e.g., `build`) and scene options (e.g., `model`)
- `MainLoop::step`
  - `do_benchmark` enters the mainloop and keeps calling `step`
  - on scene begin,
    - `before_scene_setup` deletes `fps_renderer_` and `title_renderer_`
    - `CanvasGeneric::reset`
      - reset gl state
      - create a new window
      - create a new context and make current
      - ensure an fbo if offscreen
      - `glViewport`, `glClear`, etc.
    - `Benchmark::setup_scene` loads resources, builds vbos, etc.
    - `after_scene_setup` checks if `show-fps` and `title` options are set
      - they display title and fps on screen
    - `log_scene_info` prints scene info to stdout
  - `MainLoop::draw`
    - `CanvasGeneric::clear` to clear
    - `scene_->draw` to render
    - `scene_->update` to update state
    - `CanvasGeneric::update` to swap or `glFinish`
  - on scene end,
    - `Benchmark::teardown_scene` to unload resources, etc.
    - `Scene::average_fps` calculates the fps which is added to `score_`
      - fps is calculated as `currentFrame_ / realTime_.elapsed()`
    - `log_scene_result` prints fps, etc to stdout
- scenes
  - `buffer` updates and draws a 100x20 mesh
    - `update-method` is `map` by default which causes stalls
  - `build` draws a rotating model
  - `bump` draws a rotating model with normal mapping
  - `desktop` draws a few textured rectangles
  - `effect2d` draws an image with a convolution kernel
  - `clear` draws nothing
  - `ideas` draws "Ideas in Motion" animation from 90's
  - `jellyfish` draws a translucent animating jellyfish
  - `pulsar` draws "pulsar" xscreensaver
  - `refract` draws a model with refraction
  - `shading` draws a model with gouraud shading by default
  - `shadow` draws a model with shadow mapping
  - `terrain` draws a terrain with a heightmap
  - `texture` draws a textured cube
  - `conditionals` draws a 32x32 grid whose vs/fs has `if`s
  - `function` draws a 32x32 grid whose vs/fs has a function
  - `loop` draws a 32x32 grid whose vs/fs has a loop

## vkmark

- <https://github.com/vkmark/vkmark>
- build
  - `pacman -S glm assimp wayland-protocols vulkan-headers`
  - `meson setup out -Dbuildtype=release -Dwayland=true`
- run
  - `./out/vkmark --winsys-dir out/src --data-dir data`
    - `--winsys wayland` or `--winsys headless`

## clpeak

- `device_info_t`
  - `maxAllocSize` is `CL_DEVICE_MAX_MEM_ALLOC_SIZE` (e.g., 4GB)
  - `maxGlobalSize` is `CL_DEVICE_GLOBAL_MEM_SIZE` (e.g., 32GB)
  - `maxWGSize` is `CL_DEVICE_MAX_WORK_ITEM_SIZES[0]` (e.g., 512)
  - `numCUs` is `CL_DEVICE_MAX_COMPUTE_UNITS` (e.g., 96)
  - `globalBWMaxSize` is hardcoded to 512MB
  - `computeWgsPerCU` is hardcoded to 2048
- `clPeak::runGlobalBandwidthTest`
  - `global_bandwidth_v1_local_offset` kernel
    - it sums `FETCH_PER_WI` values, each `get_local_size` apart, and writes
      the result
  - `global_bandwidth_v1_global_offset` kernel
    - it sums `FETCH_PER_WI` values, each `get_global_size` apart, and writes
      the result
    - that's bogus
- `clPeak::runComputeSP`
  - `compute_sp_v1` kernel
    - it computes `mad` with `get_local_id` for 1024 times and writes the
      result
  - `globalSize` is scaled by `numCUs` and `maxWGSize`
- `clPeak::runComputeInteger`
  - `compute_integer_v1` kernel
    - it computes `a*b+c` with `get_local_id` for 1024 times and writes the
      result
  - `globalSize` is scaled by `numCUs` and `maxWGSize`

## Vulkan Samples

- build
  - `git clone --recurse-submodules https://github.com/KhronosGroup/Vulkan-Samples.git`
  - `cd Vulkan-Samples`
  - `cmake -H. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug`
    - if wayland, `-DVKB_WSI_SELECTION=WAYLAND -DGLFW_BUILD_X11=OFF`
  - `ninja -C out`
  - `ln -sf out/app/bin/Debug/x86_64/vulkan_samples`
- deploy
  - `tar --zstd -hcf vulkan-samples.tar.zst vulkan_samples assets shaders --transform='s,,vulkan-samples/,'`
- main loop
  - stack
    - `DescriptorManagement::update`
    - `vkb::Application::step`
      - app here is a `DescriptorManagement`
    - `vkb::VulkanSamples::update`
    - `vkb::Application::step`
      - app here is a `VulkanSamples` (not `VulkanSample`)
    - `vkb::Platform::run`
    - `vkb::Platform::main_loop`
  - when the window is out of focus and is not in benchmark mode,
    `vkb::Platform::run` returns immediately and the test will be in a busy
    loop
- swapchain
  - stack of image acquire
    - `vkb::Swapchain::acquire_next_image`
    - `vkb::RenderContext::begin_frame`
    - `vkb::RenderContext::begin`
    - `DescriptorManagement::update`
  - stack of image present
    - `vkb::Queue::present`
    - `vkb::RenderContext::end_frame`
    - `vkb::RenderContext::submit`
    - `vkb::RenderContext::submit`
    - `DescriptorManagement::update`

## glbench

- <https://chromium.googlesource.com/chromiumos/platform/glbench>
  - it is really outdated
- build
  - `apt install libgflags-dev libwaffle-dev waffle-utils`
  - `make`
    - by default, it uses `PLATFORM_GLX` with big gl
    - `(cd src && USE=X GRAPHICS_BACKEND=OPENGLES make ../glbench)` to use `PLATFORM_X11_EGL` with gles
- use
  - `-list` to list all tests
  - `-tests` to specify a colon-separated list of tests
  - `-notemp` to skip temperature check
- internals
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
  - `WaffleInterface`
    - on cros, glbench uses an unofficial outdated "null" version of waffle
      plus cros's own patches
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
  - Waffle NULL platform
    - outdated
    - it uses EGL on GBM, which is also known as "surfaceless"
    - `wnull_platform_create` calls `wgbm_platform_init` and then override
      `EGL_PLATFORM` and vtbl
    - `wnull_display_connect` scans `/dev/dri/card%d` and uses the first node
      that has connectors
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

## Unity: Boat Attack

- <https://github.com/Unity-Technologies/BoatAttack>
- adb
  - `adb shell am start --windowingMode 1 -n com.UnityTechnologies.BoatAttack/com.unity3d.player.UnityPlayerActivity`
    - optional `-e unity \"-quality:hq\"`
  - `adb shell am force-stop com.UnityTechnologies.BoatAttack`

## 3DMark

- <https://benchmarks.ul.com/downloads/3dmark-android-pro.zip>
  - `adb install -r -g foo.apk`
  - `adb shell am start -n com.futuremark.dmandroid.application/com.futuremark.flamenco.ui.splash.SplashPageActivity`
  - `sleep 10`
  - `adb shell am force-stop com.futuremark.dmandroid.application`
  - `adb push foo.dlc /data/media/10/Android/data/com.futuremark.dmandroid.application/files`
  - `adb push foo.xml /data/media/10/Android/data/com.futuremark.dmandroid.application/files`
    - after editing `license_key`
  - `adb shell am start -n com.futuremark.dmandroid.application/com.futuremark.dmandroid.application.BenchmarkLauncherActivity \
       -e com.futuremark.android.InstallDLC true \
       -e com.futuremark.android.BenchmarkFilePath /storage/emulated/10/Android/data/com.futuremark.dmandroid.application/files/foo.xml`

## GravityMark

- <https://gravitymark.tellusim.com/>
  - Android
  - Linux x86-64 and arm64

## Basemark GPU

- basemarkgpu-1.2.3
- run the custom benchmark and `ps -ef` shows

    ./resources/binaries/BasemarkGPU_vk \
        TestType Custom \
        TextureCompression bc7 \
        RenderPipeline simple \
        RenderResolution 1280x720 \
        LoopCount 1 \
        GpuIndex 0 \
        ProgressBar true \
        AssetPath ./resources/assets/pkg \
        StoragePath ./logs \
        SkipZPrepass true
- `RenderPipeline` can be `simple`, `medium`, or `highend`

## Unigine Benchmarks

- <https://benchmark.unigine.com/>
- Sanctuary, 2007, 28MB
- Tropics, 2008, 56MB
- Heaven, 2009, 273MB
  - `Unigine_Heaven-4.0`
  - run the benchmark and `ps -ef` shows

      ./heaven_x64 \
          -project_name Heaven \
          -data_path ../ \
          -engine_config ../data/heaven_4.0.cfg \
          -system_script heaven/unigine.cpp \
          -sound_app openal \
          -video_app opengl \
          -video_multisample 0 \
          -video_fullscreen 0 \
          -video_mode 3 \
          -extern_define RELEASE,LANGUAGE_EN,QUALITY_HIGH,TESSELLATION_DISABLED \
          -extern_plugin GPUMonitor
  - `cat /proc/<pid>/environ | tr '\0' '\n'` shows
    - `LD_LIBRARY_PATH=x64`
  - windows version works under wine
- Valley, 2013, 394MB
  - `Unigine_Valley-1.0`
  - run the benchmark and `ps -ef` shows

      ./valley_x64 \
          -project_name Valley \
          -data_path ../ \
          -engine_config ../data/valley_1.0.cfg \
          -system_script valley/unigine.cpp \
          -sound_app openal \
          -video_app opengl \
          -video_multisample 0 \
          -video_fullscreen 1 \
          -video_mode -1 \
          -video_height 720 \
          -video_width 1280 \
          -extern_define ,RELEASE,LANGUAGE_EN,QUALITY_HIGH \
          -extern_plugin ,GPUMonitor
- Superposition, 2017, 1564MB
