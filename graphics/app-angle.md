ANGLE
=====

## Build and Use

- <https://chromium.googlesource.com/angle/angle/+/HEAD/doc/DevSetup.md>
  - `git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git`
  - `export PATH=`pwd`/depot_tools:$PATH`
  - `git clone https://chromium.googlesource.com/angle/angle`
  - `cd angle`
  - `python scripts/bootstrap.py`
  - `gclient sync`
  - `sudo ./build/install-build-deps.sh`
  - `gn gen out/Debug`
  - `autoninja -C out/Debug`
- Run
  - export `LD_LIBRARY_PATH=<path-to-angle>/out/Debug`
  - `ldd es2gears`
  - `es2gears`
  - `ANGLE_DEFAULT_PLATFORM=vulkan es2gears`
    - this does not work because the native window is considered invalid
      - iirc, `xcb_create_window` != `XCreateWindow` and angle expects xcb
    - glmark2 with `-Dflavors=x11-glesv2` works
- Android
  - <https://chromium.googlesource.com/angle/angle/+/HEAD/doc/DevSetupAndroid.md>
  - `echo "target_os = ['android']" >> .gclient`
  - `gclient sync`
  - `gn args out/Android`
    - `target_os = "android"`
    - `target_cpu = "arm64"`
    - `angle_enable_vulkan = true`
    - `angle_expose_non_conformant_extensions_and_versions = true`
    - `is_component_build = false`
    - `is_official_build = true`
    - `is_debug = false`
    - `angle_assert_always_on = false`
    - `use_goma = true`
  - `autoninja -C out/Android`
  - `adb install -r -d --force-queryable out/Android/apks/AngleLibraries.apk`
    - verify installation with `adb shell pm path org.chromium.angle`
  - enable angle
    - `adb shell settings put global angle_debug_package org.chromium.angle`
    - `adb shell settings put global angle_gl_driver_all_angle 1`
  - disable
    - `adb shell settings delete global angle_gl_driver_all_angle`
    - `adb shell settings delete global angle_debug_package`
  - to verify, `adb logcat -d | grep ANGLE`
- cross compile
  - `./build/linux/sysroot_scripts/install-sysroot.py --arch=arm64`
  - `gn args out/AArch64`, same as android except
    - `target_os = "linux"`
    - `is_official_build = false`
  - `autoninja -C out/AArch64`
  - package
    - `cd out`
    - `angledir=angle-aarch64-$(date +%Y%m%d)`
    - `find out/AArch64 -maxdepth 1 -type f -executable | xargs aarch64-linux-gnu-strip -x`
    - `find out/AArch64 -maxdepth 1 -type f -executable -o -name gen | xargs tar cf $angledir.tar --transform="s,out/AArch64,$angledir,"`
    - `tar rf $angledir.tar --transform="s,,$angledir/," src/tests/deqp_support/*.txt third_party/VK-GL-CTS/src/android/cts/main/*.txt third_party/VK-GL-CTS/src/external/openglcts/data/mustpass/gles/aosp_mustpass/main/*.txt`
    - `zstd $angledir.tar`
    - if want to use on regular egl/gles apps with `LD_LIBRARY_PATH`,
      - `ln -sf libEGL.so libEGL.so.1`
      - `ln -sf libGLESv2.so libGLESv2.so.2`
  - `angle_deqp_egl_tests`
    - use `strace -e trace=file` to figure out what files are missing
    - `ls third_party/VK-GL-CTS/src/android/cts/main/*-master.txt \
       src/tests/deqp_support/*_test_expectations.txt | \
       xargs tar zcf deqp-data.tar.gz`
    - `--deqp-egl-display-type=angle-vulkan` selects the vulkan config, which
      is used to select test expectation file?
    - `--deqp-case` converts deqp-style test names to `--gtest_filter`
    - `--renderdoc` eanbles renderdoc frame captures

## `eglGetDisplay`

- <https://chromium.googlesource.com/angle/angle/+/refs/heads/main/extensions/EGL_ANGLE_platform_angle.txt>
- stack
    #0 egl::Display::GetDisplayFromNativeDisplay
    #1 egl::GetDisplay
    #2 EGL_GetDisplay
    #3 eglGetDisplay
- `GetDisplayFromNativeDisplay`
  - platform type is `EGL_PLATFORM_ANGLE_ANGLE`
  - display type is `EGL_PLATFORM_ANGLE_TYPE_OPENGL_ANGLE`
    - when `ANGLE_DEFAULT_PLATFORM` is not set, it is mapped to
      `EGL_PLATFORM_ANGLE_TYPE_OPENGL_ANGLE`
    - `ANGLE_DEFAULT_PLATFORM=gl` maps it to
      `EGL_PLATFORM_ANGLE_TYPE_OPENGLES_ANGLE`
    - `vulkan` or `swiftshader` maps it to
      `EGL_PLATFORM_ANGLE_TYPE_VULKAN_ANGLE`
    - it can also be set programatically by apps through attrib
      `EGL_PLATFORM_ANGLE_TYPE_ANGLE`
  - device type is `0`
    - it is mapped to `EGL_PLATFORM_ANGLE_DEVICE_TYPE_HARDWARE_ANGLE`
    - `ANGLE_DEFAULT_PLATFORM=swiftshader` maps it to
      `EGL_PLATFORM_ANGLE_DEVICE_TYPE_SWIFTSHADER_ANGLE`
    - it can also be set programatically by apps through attrib
      `EGL_PLATFORM_ANGLE_DEVICE_TYPE_ANGLE`
  - platform type is `0`
    - it is mapped to `EGL_PLATFORM_X11_EXT` on Linux/X11
    - it can also be set programatically by apps through attrib
      `EGL_PLATFORM_ANGLE_NATIVE_PLATFORM_TYPE_ANGLE`

## `eglInitialize`

- stack
    #0  rx::RendererVk::initialize
    #1  rx::DisplayVk::initialize
    #2  rx::DisplayVkXcb::initialize
    #3  egl::Display::initialize
    #4  egl::Initialize
    #5  EGL_Initialize
    #6  eglInitialize

## Android

- `GraphicsEnvironment.java`
  - `getDriverForPackage`
    - if `ANGLE_GL_DRIVER_ALL_ANGLE`, return `angle`
    - else check `ANGLE_GL_DRIVER_SELECTION_PKGS` and
      `ANGLE_GL_DRIVER_SELECTION_VALUES`
  - `shouldUseAngle` returns true when `getDriverForPackage` returns `angle`
  - `getAngleDebugPackage` returns `ANGLE_DEBUG_PACKAGE`
  - `setAngleInfo` passes angle info to native code

## Dispatch

- EGL
  - public symbols, such as `eglFoo`, are defined in
    `libEGL/libEGL_autogen.cpp`
  - `eglFoo` calls into `EGL_Foo` defined in
    `libGLESv2/entry_points_egl_autogen.cpp`
  - most `EGL_Foo` call into `egl::Foo` defined in `libGLESv2/egl_stubs.cpp`
- GLESv1
  - public symbols, such as `glFoo`, are defined in
    `libGLESv1_CM/libGLESv1_CM.cpp`
  - `glFoo` calls into `GL_Foo` defined in
    `libGLESv2/entry_points_gles_1_0_autogen.cpp`
  - `GL_Foo` calls into `Context::foo` defined in
    `libANGLE/Context_gles_1_0.cpp`
- GLESv2
  - public symbols, such as `glFoo`, are defined in
    `libGLESv2/libGLESv2_autogen.cpp`
  - `glFoo` calls into `GL_Foo` defined in
    `libGLESv2/entry_points_gles_2_0_autogen.cpp`
  - `GL_Foo` calls into `Context::foo` defined in
    `libANGLE/Context.cpp`

## Command Recording

- `glClear` call sequence
  - `GL_Clear`
  - `gl::Context::clear`
  - `rx::FramebufferVk::clear`
  - `rx::FramebufferVk::clearImpl`
- `glDrawElements` call sequence
  - `GL_DrawElements`
  - `gl::Context::drawElements`
  - `rx::ContextVk::drawElements`
- `glFlush` call sequence
  - `GL_Flush`
  - `gl::Context::flush`
  - `rx::ContextVk::flush`
  - `rx::ContextVk::flushImpl`
  - `rx::ContextVk::flushAndGetSerial`
- commands are recorded to `mOutsideRenderPassCommands` or
  `mRenderPassCommands`
  - they are only recorded
  - they are executed (i.e., call into the vk driver) on flush
- on first draw
  - `rx::ContextVk::setupDraw`
  - `rx::ContextVk::handleDirtyGraphicsRenderPass`
  - `rx::ContextVk::startRenderPass`
  - `rx::FramebufferVk::startNewRenderPass`
  - `rx::ContextVk::beginNewRenderPass`
  - `rx::vk::RenderPassCommandBufferHelper::beginRenderPass`
- on flush
  - `rx::ContextVk::flushCommandsAndEndRenderPassImpl`
  - `rx::vk::RenderPassCommandBufferHelper::endRenderPass`

## Traces

- <https://chromium.googlesource.com/angle/angle/+/HEAD/src/tests/restricted_traces/README.md>
- data files
  - `angle_trace_tests` uses `src/tests/restricted_traces` as the data dir
  - `restricted_traces.json` lists all traces
  - `$trace/$trace.json` has the metadata for `$trace`
- `angle_trace_tests`
  - `--gtest_list_tests` lists all tests
    - e.g., `TraceTest.pokemon_go`
    - `RegisterTraceTests` parses trace jsons and generates them dynamically
  - `--gtest_filter` to filter tests
  - `ANGLEProcessPerfTestArgs` understands these options
    - `--use-gl=angle` selects `GLESDriverType::AngleEGL` and loads `libEGL.so`
    - `--use-gl=system` selects `GLESDriverType::SystemEGL` and loads `libEGL.so.1`
    - `--use-angle=vulkan` selects the vulkan backend when using angle
    - `--minimize-gpu-work` forces 1x1 window
    - `--verbose`
    - `--offscreen`
- `TracePerfTest` is the class to run tests
  - `ANGLERenderTest::SetUp`
    - creates an `OSWindow`
    - calls `initializeGLWithResult` to initialize EGL/GLES
    - calls `TracePerfTest::initializeBenchmark` to initialize trace loader
    - `iterationsPerStep` is 1
    - calls `ANGLEPerfTest::runTrial` to warm up
      - `gWarmupTrials` is 3 and `gCalibrationTimeSeconds` is 1.0 by default
      - it keeps `ANGLERenderTest::step` and `TracePerfTest::drawBenchmark`
      	until `gCalibrationTimeSeconds` is reached
      - and repeat above for `gWarmupTrials` times
      - each step draws a frame of the trace
- outputs
  - `cpu_time` is average cpu time (could be larger than `wall_time` when threaded)
    per step
  - `wall_time` is average `CLOCK_MONOTONIC` per step
  - `trial_steps` how many steps (frames) in this trial
  - `total_steps` how many steps in `gTestTrials` trials
  - `memory_median`
  - `memory_max`
- android
  - push traces to `/sdcard/chromium_tests_root/src/tests/restricted_traces`
  - `adb shell appops set --uid com.android.angle.test MANAGE_EXTERNAL_STORAGE 0`
  - `adb shell appops write-settings`
  - `adb shell am instrument -w \
       -e org.chromium.native_test.NativeTestInstrumentationTestRunner.StdoutFile /sdcard/chromium_tests_root/out.txt \
       -e org.chromium.native_test.NativeTest.CommandLineFlags '--gtest_filter=TraceTest.*' \
       -e org.chromium.native_test.NativeTestInstrumentationTestRunner.ShardNanoTimeout 1000000000000000000 \
       -e org.chromium.native_test.NativeTestInstrumentationTestRunner.NativeTestActivity com.android.angle.test.AngleUnitTestActivity \
       com.android.angle.test/org.chromium.build.gtest_apk.NativeTestInstrumentationTestRunner`
  - <https://chromium.googlesource.com/chromium/src/+/main/testing/android/docs/gtest_implementation.md>
    - `gclient sync` clones `testing` from chromium
    - it builds an apk containing
      - one or more `.so` 
      - one or more `.dex` 
      - a manifest file containing `<instrument>` and `<activity>`

## `GL_KHR_blend_equation_advanced`

- if `VK_EXT_rasterization_order_attachment_access` and
  `rasterizationOrderColorAttachmentAccess`,
  - `supportsRasterizationOrderAttachmentAccess` is set
  - `supportsShaderFramebufferFetch` is set
  - `GL_EXT_shader_framebuffer_fetch` is advertised
- if `VK_EXT_blend_operation_advanced`,
  - `supportsBlendOperationAdvanced` is set
  - otherwise, `emulateAdvancedBlendEquations` is set if not on intel
  - `GL_KHR_blend_equation_advanced` is advertised if either is set
