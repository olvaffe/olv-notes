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
  - `ninja -C out`
  - `ln -sf out/app/bin/Debug/x86_64/vulkan_samples`
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
- cmdline
  - `-w 1920 -h 1080 --gfx egl -t gl_trex_off,gl_trex`
  - `--ei -single_frame=20000`
  - `adb shell am broadcast -a net.kishonti.testfw.ACTION_RUN_TESTS -n
    net.kishonti.gfxbench.v50004.corporate/net.kishonti.benchui.corporate.CommandLineSession
    -e test_ids 'gl_manhattan31' --ei raw_config.single_frame 20000`
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
