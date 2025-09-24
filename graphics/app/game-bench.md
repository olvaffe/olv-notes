GPU Benchmarks
==============

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

## Unigine Heaven

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

## Unigine Valley

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

## GravityMark

- <https://gravitymark.tellusim.com/>
  - Android
  - Linux x86-64 and arm64

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

## GFXBench

- results
  - `elapsed_time`, total time in animation?
  - `load_time`, time to load resources?
  - `measured_time`, same as `elapsed_time`?
  - `frame_count`, frame rendered
  - `score`, same as `frame_count`?
  - `fps`, `frame_count` divided by `measured_time`?
- played with `gl_trex.json` a bit
  - trex is an animation of length 56s.  It knows the scene to render at any
    point in time of the animation.
    - after 56s, it keeps rendering the last scene but water is still
      flowing, etc.
  - `[start_animation_time, play_time]` is the begin/end times of the
    animation
  - `single_frame` renders the scene at the specified time indefinitely
  - `frame_step_time` skips forward the specified ms after rendering each
    scene
    - with play time 1000 and step time 20, it will render 50 frames
  - `loop_count` repeats the test N times
  - to change off-screen test sizes,
    - set `test_width` and `test_height`
    - but better set `offscreen_width` and `offscreen_height` as well to be
      safe
- linux cmdline
  - `-w 1920 -h 1080 --gfx egl -t gl_trex_off,gl_trex`
  - `--ei -single_frame=20000`
- android cmdline
  - `adb install -r -g gfxbench_vulkan-5.0.5+corporate.apk`
  - `adb shell am start net.kishonti.gfxbench.vulkan.v50005.corporate/net.kishonti.app.MainActivity`
  - `adb shell am broadcast -a net.kishonti.testfw.ACTION_RUN_TESTS \
       -n net.kishonti.gfxbench.v50005.corporate/net.kishonti.benchui.corporate.CommandLineSession \
       -e test_ids 'gl_manhattan31,gl_trex' --ei raw_config.single_frame 20000`
  - `adb pull /sdcard/Android/data/net.kishonti.gfxbench.vulkan.v50005.corporate/files/results`
- test configs from android apk
  - 3.0
    - `gl_alu`, ALU
      - Measures the pure shader compute performance of your device by using a
        complex fragment shader and rendering a single full-screen quad.
    - `gl_alu_off`, 1080p ALU Offscreen
      - `"play_time": 30000`, length of test in ms
      - `""screenmode": 1`, on-screen or off-screen
    - `gl_blending`, Alpha Blending
      - Measures the device's alpha blending performance by rendering
        semi-transparent screen-aligned quads with high-resolution
        uncompressed textures.
    - `gl_blending_off`, 1080p Alpha Blending Offscreen
    - `gl_driver`, Driver Overhead
      - Measures the OpenGL driver’s CPU overhead by rendering a large number
        of simple objects one-by-one, changing device state for each item. The
        frequency of the state changes reflects real-world applications.
    - `gl_driver_off`, 1080p Driver Overhead Offscreen
    - `gl_fill`, Fill
      - Measures the device’s texturing performance by rendering four layers
        of compressed textures. This is a common scenario in gaming.
    - `gl_fill_off`, 1080p Fill Offscreen
    - `gl_manhattan`, Manhattan
      - This is the original Manhattan test, first introduced in GFXBench 3.0,
        which uses the ES 3.0 / GL 4.1 capabilities of your device. The test
        scene is a night-time city environment with lots of lights to
        illuminate the scenery. It uses deferred rendering and multi-render
        target (MRT) for the geometry pass (four color attachments as
        textures), diffuse and specular lighting and features complex effects
        such as depth shadow map, cloaking effect, post-process effects
        (bloom, depth of field), occlusion query and animated volumetric light
        shafts.
    - `gl_manhattan_off`, 1080p Manhattan Offscreen
    - `gl_manhattan_fixed_off`, 1080p Manhattan Fixed Time Offscreen
      - `"frame_step_time": 40`, render 1 frame per specified ms?
    - `gl_trex`, T-Rex
      - This is the original T-Rex test, first introduced in GFXBench 2.7.
        Based on ES 2.0 / GL 3.0, the T-Rex test includes highly detailed
        textures, materials, complex geometry, particles with animated
        textures and a post-process effect: motion blur. The graphics pipeline
        also features effects, e.g.: planar reflection, specular highlights
        and soft-shadows.
    - `gl_trex_off`, 1080p T-Rex Offscreen
    - `gl_trex_fixed_off`, 1080p T-Rex Fixed Time Offscreen
    - `gl_trex_battery`, Battery Test - T-Rex
      - This is the original Battery Test, first introduced in GFXBench 3.0.
        This combined test measures your device’s battery life and FPS
        stability in a gaming scene by running the same high-level test for 30
        times. It logs FPS and the battery charge during the test and the
        results are displayed in a battery graph. The test produces two
        scores, one is the estimated battery life in minutes while the other
        is the number of frames rendered by the slowest test run.
      - `"loop_count": 30`
      - `"battery_charge_drop": -1`
      - `"battery": true`
    - `gl_trex_qmatch`, Render Quality
      - This is the original Render Quality test, first introduced in GFXBench
        3.0. It measures the visual fidelity of your device in a high-end
        gaming scene. The score is the peak signal-to-noise ratio (PSNR) based
        on mean square error (MSE) compared to a pre-rendered reference image.
      - `"quality": true`
      - `"single_frame": 23760`
    - `gl_trex_qmatch_highp`, Render Quality (high precision)
      - `"force_highp": true`
  - 3.1
    - `gl_alu2`, ALU 2
      - This is an enhanced version of the original ALU test found in GFXBench
        3.0. It approximates the fragment shader computing load of the
        Manhattan test better, by rendering 64 point lights in multiple passes
        over a captured Manhattan frame. Emissive and ambient terms are also
        evaluated along with the diffuse lighting.
    - `gl_alu2_off`, 1080p ALU 2 Offscreen
    - `gl_diag`, GL Diagnostics
    - `gl_driver2`, Driver Overhead 2
      - This is an enhanced version of the original Driver Overhead test found
        in GFXBench 3.0, and approximates the graphics driver's CPU load when
        running the Manhattan high-level test. It renders the same amount of
        geometry twice to add a glow effect, using lots of draw calls,
        changing rendertargets, vertex formats and shaders as well. The test
        also alters graphics states like depth-test and blending between
        blocks of draw calls.
    - `gl_driver2_off`, 1080p Driver Overhead 2 Offscreen
    - `gl_fill2`, Texturing
      - This is an enhanced version of the original Fill test found in
        GFXBench 3.0. It approximates the texturing load of the Manhattan
        high-level test better, by rendering multiple layers of textured
        planes over each other. The amount of textures and their formats
        matches those of Manhattan, including uncompressed screen-sized
        sources, depth-format buffers, and cube-maps.
    - `gl_fill2_off`, 1080p Texturing Offscreen
    - `gl_manhattan31`, Manhattan 3.1
      - This is an enhanced version of the original Manhattan test found in
        GFXBench 3.0, showcasing advanced visual effects achieved with OpenGL
        ES 3.1 / OpenGL 4.3. Compute shaders and indirect draw calls are used
        for the lightning effect, for the HDR with luminance adaptation, and
        the bloom thresholding as well. The depth-of-field effect is also
        enhanced.
    - `gl_manhattan31_off`, 1080p Manhattan 3.1 Offscreen
    - `gl_manhattan31_fixed_off`, 1080p Manhattan 3.1 Fixed Time Offscreen
    - `gl_manhattan31_battery`, Battery Test - Manhattan 3.1
    - `gl_manhattan311_wqhd_off`, 1440p Manhattan 3.1.1 Offscreen
      - `"test_width": 2560`, test width
      - `"test_height": 1440`, test height
      - `"offscreen_width": 2560`, for backward compat
      - `"offscreen_height": 1440`, for backward compat
    - `gl_manhattan311_fixed_wqhd_off`, 1440p Manhattan 3.1.1 Fixed Time Offscreen
  - 4.0
    - `gl_4`, Car Chase
      - Car Chase is our game-like test that fully exploits the capabilities
        of the Android Extension Pack. It uses deferred shading combined by
        image-based lighting to render the scene in a full HDR environment.
        The test uses geometry shaders and tessellation, dynamic reflections,
        physically-based shading, compute based particles and occlusion
        culling, dynamic cascaded shadows and advanced post-process effects,
        like motion blur, screen-space ambient occlusion, depth of field and
        lens flare.
    - `gl_4_off`, 1080p Car Chase Offscreen
    - `gl_4_fixed_off`, 1080p Car Chase Fixed Time Offscreen
    - `gl_tess`, Tessellation
      - Measures the device's tessellation performance by rendering multiple
        quadratic Bézier surface based torus knot models.
    - `gl_tess_off`, 1080p Tessellation Offscreen
  - 5.0
    - `gl_5_high`, Aztec Ruins OpenGL (High Tier)
      - Aztec Ruins is our cross API game-like test available for most modern
        graphics APIs. It features real-time global illumination over deferred
        rendering combined with physically-based shading. It uses a variety of
        popular rendering algorithms such as occlusion culling, dynamic
        cascaded shadows and advanced post-process effects, like motion blur,
        screen-space ambient occlusion, depth of field. The High Tier version
        features particle simulation and higher quality illumination and
        effects over Normal Tier.
    - `gl_5_high_off`, 1440p Aztec Ruins OpenGL (High Tier) Offscreen
    - `gl_5_normal`, Aztec Ruins OpenGL (Normal Tier)
      - `"scenefile": "scene5_normal/scene_5.xml"`
    - `gl_5_normal_off`, 1080p Aztec Ruins OpenGL (Normal Tier) Offscreen
    - `vulkan_5_high`, Aztec Ruins Vulkan (High Tier)
      - `"test_id": "vulkan_5_high"`
    - `vulkan_5_high_off`, 1440p Aztec Ruins Vulkan (High Tier) Offscreen
    - `vulkan_5_normal`, Aztec Ruins Vulkan (Normal Tier)
    - `vulkan_5_normal_off`, 1080p Aztec Ruins Vulkan (Normal Tier) Offscreen

## GFXBench `gl_manhattan31`

- `-single_frame=56000`
- a depth-only pass for the robot
  - outputs
    - texture 462, 1024x1024, `D16`
- draw the scene to gbuffers
  - outputs
    - texture 348-351, 1920x1080, `R8G8B8A8_UNORM`
    - texture 347, 1920x1080, `D24`
- a compute pass to calculate vertices of lightning and point lights
- bounding box checks
  - draw 5 points and query for `GL_ANY_SAMPLES_PASSED`
  - no color/depth write
- draw point lights
  - inputs
    - gbuffers
  - outputs
    - texture 352, 1920x1080, `R10G10B10A2_UNORM`
- lighting pass
  - inputs
    - texture 352, 1920x1080, `R10G10B10A2_UNORM`
    - gbuffers
  - outputs
    - texture 373, 1920x1080, `R10G10B10A2_UNORM`
- draw translucent objects (e.g., lights of light sources)
  - a compute pass to calculate more vertices of lightning
    - inputs
      - buffer 860
    - outputs
      - buffer 859
  - draw lightning
    - inputs
      - buffer 859
    - outputs
      - texture 373, 1920x1080, `R10G10B10A2_UNORM`
  - several draws
    - inputs
      - various textures
    - outputs
      - texture 373, 1920x1080, `R10G10B10A2_UNORM`
- generate bloom texture
  - a compute pass
    - calculate regional brightness
      - inputs
        - texture 373, 1920x1080, `R10G10B10A2_UNORM`
      - outputs
        - buffer 903
      - for every 16x8 region, calculate its average brightness
    - calculate global brightness
      - inputs
        - buffer 903
      - outputs
        - buffer 904
    - apply tone mapping
      - inputs
        - texture 373, 1920x1080, `R10G10B10A2_UNORM`
        - buffer 904
      - outputs
        - texture 909, 960x540, `R8G8B8A8_UNORM`, 4 mips
  - generate mipmap
    - inputs/outputs
      - texture 909, 960x540, `R8G8B8A8_UNORM`, 4 mips
  - apply vertical gaussian blur
    - inputs
      - texture 909, 960x540, `R8G8B8A8_UNORM`, 4 mips
    - outputs
      - texture 889, 960x540, `R8G8B8A8_UNORM`, 4 mips
    - 4 draws for 4 mips
  - apply horizontal gaussian blur
    - inputs
      - texture 889, 960x540, `R8G8B8A8_UNORM`, 4 mips
    - outputs
      - texture 894, 960x540, `R8G8B8A8_UNORM`, 4 mips
    - 4 draws for 4 mips
- post-process
  - apply tone-mapping and bloom
    - inputs
      - texture 373, 1920x1080, `R10G10B10A2_UNORM`
      - texture 894, 960x540, `R8G8B8A8_UNORM`, 4 mips
    - outputs
      - texture 375, 1920x1080, `R8G8B8A8_UNORM`
  - apply vertical gaussian blur
    - inputs
      - texture 375, 1920x1080, `R8G8B8A8_UNORM`
    - outputs
      - texture 878, 1920x1080, `R8G8B8A8_UNORM`
  - apply horizontal gaussian blur
    - inputs
      - texture 878, 1920x1080, `R8G8B8A8_UNORM`
    - outputs
      - texture 880, 1920x1080, `R8G8B8A8_UNORM`
  - apply DoF
    - inputs
      - texture 375, 1920x1080, `R8G8B8A8_UNORM`
      - texture 880, 1920x1080, `R8G8B8A8_UNORM`
      - texture 347, 1920x1080, `D24`
    - outputs
      - back color
      - back depth (not used)
- fd renderstages @ 2574x1380, frame time 95ms
  - Compute
    - 0.3ms
  - Surface: z16, 1024x1024
    - 0.9ms
  - Surface: r8g8b8a8 x4 + z24, 2574x1380
    - 13.6ms
      - bin 224x96, count 180=12x15
      - binning 2.4ms
  - Surface: r10g10b10a2 + z24, 2574x1380
    - 19.4ms
  - Surface: r10g10b10a2 + z24, 2574x1380
    - 6.0ms
  - Surface: r10g10b10a2 + z24, 2574x1380
    - 6.0ms
  - Compute
    - 0.2ms
  - Surface: r10g10b10a2 + z24, 2574x1380
    - 9.5ms
  - Compute
    - 2.4ms
  - Compute
    - 4.8ms
  - Surface: r8g8b8a8, 1287x690, mipmaped
    - 1.9ms, 0.5ms, 0.1ms, 0.0ms
  - Surface: r8g8b8a8, 1287x690, mipmaped
    - 1.7ms, 0.5ms, 0.1ms, 0.0ms
  - Surface: r8g8b8a8, 2574x1380
    - 3.8ms
  - Surface: r8g8b8a8, 2574x1380
    - 6.8ms
  - Surface: r8g8b8a8, 2574x1380
    - 6.0ms
  - Surface: r8g8b8x8 + z24, 2574x1380
    - 3.5ms
- tu renderstages @ 2574x1380, frame time 105ms
  - command buffer, 21.3ms
    - Render Pass: z16, 1024x1024
      - 0.9ms
    - Compute
      - 0.3ms
    - Render Pass: r8g8b8a8 x4 + z24, 2574x1380
      - 20.2ms
      - bin 160x128, count 187=17x11
      - binning 3.1ms
  - command buffer, 20.6ms
    - Render Pass: r10g10b10a2 + z24, 2574x1380
      - 1.1ms
    - Render Pass: r10g10b10a2 + z24, 2574x1380
      - 19.5ms
  - command buffer, 51.2ms
    - Compute
      - 0.2ms
    - Render Pass: r10g10b10a2 + z24, 2574x1380
      - 16.1ms
    - Compute
      - 2.3ms
    - Compute
      - 1.1ms
    - Render Pass: r8g8b8a8, 1287x690, mimapped
      - 2.7ms, 0.7ms, 0.2ms, 0.1ms
    - Render Pass: r8g8b8a8, 1287x690, mimapped
      - 2.4ms, 0.6ms, 0.2ms, 0.1ms
    - Render Pass: r8g8b8a8, 2574x1380
      - 3.8ms
    - Render Pass: r8g8b8a8, 2574x1380
      - 9.5ms
    - Render Pass: r8g8b8a8, 2574x1380
      - 7.7ms
    - Render Pass: r8g8b8a8, 2574x1380
      - 3.2ms

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

## Unity Boat Attack

- adb
  - `adb shell am start -n com.UnityTechnologies.BoatAttack/com.unity3d.player.UnityPlayerActivity`
  - `adb shell am force-stop com.UnityTechnologies.BoatAttack`
