GFXBench
========

## Build

- <https://github.com/Kishonti-Opensource/gfxbench>
- build
  - `export PRODUCT_ID=gfxbench_vulkan`
    - this implies gl and vk, while the default `gfxbench` implies dx as well
  - `export PLATFORM=linux`
  - `export CONFIG=Release`
  - `export COMMON_OPTS="-GNinja -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache"`
  - `export VULKAN_LIB_PATH=$(pkg-config --variable=libdir vulkan)`
    - this helps ngl finds the right vulkan loader
  - `scripts/build-3rdparty.sh`
    - build fixes
      - edit `3rdparty/zlib/CMakeLists.txt` to always set `Z_HAVE_UNISTD_H`
      - edit `3rdparty/poco/CMakeLists.txt` to default `DISABLE_OPENSSL` to `ON`
  - `scripts/build.sh`
    - build fixes
      - edit `scripts/build.sh` to remove `frameworks/cudaw`
- dist
  - `tar --zstd -cf tfw-pkg.tar.zst tfw-pkg`
- run
  - `./bin/testfw_app --gfx=egl -w 1024 -h 768 -t gl_5_normal`
  - `./bin/testfw_app --gfx=xcb_vulkan -w 1024 -h 768 -t vulkan_5_normal`
- `scripts/build-3rdparty.sh` builds these projects using cmake under `out/`
  - `3rdparty/libepoxy`
  - `3rdparty/zlib`
  - `3rdparty/libpng`
  - `3rdparty/poco`
  - `3rdparty/AgilitySDK`
  - `3rdparty/glew` because of `PLATFORM=linux`
  - `3rdparty/glfw` because of `PLATFORM=linux`
  - `frameworks/ngl/src/v1.0.3/loader` because of `PLATFORM=linux`
- `scripts/build.sh` builds these projects using cmake under `out/`
  - `frameworks/ngrtl` for basic utilities
  - `gfxbench-data`
    - `/gfxbench-data40` for 2.0, 3.0, 4.0 assets
    - `/gfxbench-data50` for 5.0 assets
  - `frameworks/clew` similar to glew but for CL
  - `frameworks/systeminfo` for sysinfo collection
  - `frameworks/oglx` provides `oglx/gl.h` to include the real gl headers
  - `frameworks/testfw`
    - `/gfxbench`
      - `/gfxbench40`
        - `/frameworks/kcl_framework`
        - `/frameworks/testfw/schemas`
        - `src/common`
        - `src/tests/40`
          - `../../gfx4`
            - `../common_31`
            - `../gfx3_1`
              - `../gfx3_0`
                - `/frameworks/krl`
      - `/gfxbench50`
        - `/frameworks/kcl_framework`
        - `/frameworks/ngl`
        - `src/testbases`
          - `/frameworks/ksl_compiler`
        - `src/common`
          - `/frameworks/ksl_compiler`
        - `src/scene5`
    - `deviceinfo` for devinfo collection
    - `resourcemanager` unused
    - `hostapp/basic` provides `testfw_app` executable

## Cross-Compile

- build
  - `export PLATFORM=linux_arm64` instead of `export PLATFORM=linux`
  - `export OGLX_VARIANT=sys`
  - `export OGLX_DRIVER=ES3`
  - additional build fixes
    - edit `frameworks/cmake-utils/cmake/toolchain/linux_arm64.cmake` to add
      `CMAKE_SYSROOT` and `CMAKE_FIND_ROOT_PATH_MODE_*`
- dist
  - `ln -sf ../out/install/linux_arm64/bin tfw-pkg`
  - `tar --zstd -chf tfw-pkg.tar.zst tfw-pkg`
- `export OGLX_VARIANT=sys`
  - it is `default` by default, and oglx links to sys big GL and glew
  - if `sys`, oglx uses its own khr headers and links to sys egl/gles
  - if `dummy` oglx uses its own khr headers and links to dummy egl/gles
- `export OGLX_DRIVER=ES3`
  - it is empty by default, and `FindOGLX.cmake` finds glew before falling
    back to big gl
  - if `ES3`, `FindOGLX.cmake` finds egl/gles instead

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
  - `adb install -r -g gfxbench_vulkan-5.1.5+corporate.apk`
  - `adb shell am start net.kishonti.gfxbench.vulkan.v50105.corporate/net.kishonti.app.MainActivity`
  - `adb shell am broadcast -a net.kishonti.testfw.ACTION_RUN_TESTS \
       -n net.kishonti.gfxbench.vulkan.v50105.corporate/net.kishonti.benchui.corporate.CommandLineSession \
       -e test_ids 'gl_manhattan31,gl_trex' --ei raw_config.single_frame 20000`
  - `adb pull /sdcard/Android/data/net.kishonti.gfxbench.vulkan.v50105.corporate/files/results`
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

## GFXBench `gl_trex`

- `-single_frame=20000`
- `Creating test factory for: gl_trex`
  - `Runner::prepare` calls `TestFactory::test_factory` to dlsym for
    `create_test_gl_trex` and calls `TestFactory::create_test` to create the
    test
    - `create_test_gl_trex` is defined by `CREATE_FACTORY(gl_trex, GFXBenchCorporateA<Engine2>)`
- `Initializing GLES context` to `Selected EGL cofiguration`
  - `Runner::prepareGraphics` calls `WindowFactory::create` to create a window
  - with `--gfx=egl`, `WindowFactory::createEGL` creates a `XGraphicsWindow`
    and a `EGLGraphicsContext`
  - `EGLGraphicsContext::initWindowSurface` goes through `eglGetDisplay`,
    `eglInitialize`, `eglGetConfigs`, `eglCreateWindowSurface`,
    `eglCreateContext`, `eglMakeCurrent`
- `Preparing done.`
- `in loop`
  - they handle X11 events
- `Initializing gl_trex` to `Initialization successful`
  - `Runner::run` calls `GFXBenchA<Engine2>::init`
  - `OpenGL Vendor: ...` to `feature ...`
    - `GFXBench::InitializeTestEnvironment` prints gl, fb, ext info
  - `Texture type=ETC1`, `PROGRESS: ...`, to `Using mediump in fs.`
    - `GFXBenchA<Engine2>::init` creates `Engine2` and calls `InitializeTest`
      - `Engine2::Engine2` creates `GLB_Scene_ES2`
      - `TestBase::init0` calls `Engine2::init`
    - `SceneHandler::Process_Load`
      - progress 0.10 to 0.30 load scene graph
      - `trex/scene_trex.xml`
      - `animations/*` seems to specify where each object goes at any
        specific timestamp
      - `actors.txt` seems to specify objects in the scene, including their
        meshes (`meshes/*`) and materials (`materials/*`)
      - `rooms.txt` seems to specify backgrounds in the scene, very similar
        to `actors.txt`
      - `portals.txt` seems to specify how rooms are connected
      - `envmaps.txt`
    - `GLB_Scene_ES2::Process_GL`
      - progress 0.50 to 0.52 prep ubershader
      - progress 0.53 to 0.55 create scene fbos
      - progress 0.56 to 0.58 load images
      - progress 0.60 to 0.70 load meshes
      - progress 0.70 to 0.75 compile shaders
      - progress 0.75 to 0.80 create textures
- `Running gl_trex`
  - this steps through the scene until done
  - for each frame, it updates the states and renders
  - frame analysis
    - the first pass renders shadow map for the biker
    - the second pass renders shadow map for trex
    - the third pass renders the frame with `zprepass`
      - disables color mask
      - renders opaque objects (ground, tree, biker, trex, bushes, etc.) to the depth buffer
      - enables color mask
      - renders all objects
      - renders skybox
    - the fourth pass renders displacements of moving objects
      - renders pos-delta for moving objects
    - it combines fbos from 3rd and 4th passes to produce the final frame
