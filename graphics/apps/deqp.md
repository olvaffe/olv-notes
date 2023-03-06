dEQP
====

## Build

- Build
  - `python3 external/fetch_sources.py`
  - `cmake -S . -B out -G Ninja -DCMAKE_BUILD_TYPE=Debug -DDEQP_TARGET=surfaceless`
    - `DEQP_TARGET` is mostly for GL/GLES
  - `ninja`
- Package
  - `mkdir deqp-dist`
  - `cd deqp-dist`
  - `for i in vulkan/deqp-vk vulkan/vulkan; do ln -sf ../external/vulkancts/modules/$i; done`
  - `for i in egl/deqp-egl gles2/deqp-gles2 gles2/gles2 gles3/deqp-gles3 gles3/gles3 gles31/deqp-gles31 gles31/gles31; do ln -sf ../modules/$i; done`
  - `for i in vk-master.txt vk-master egl-master.txt gles2-master.txt gles3-master.txt gles31-master.txt; do ln -sf ../../android/cts/main/$i; done`
  - `strip deqp-*`
  - `cd ..`
  - `tar zchf deqp-dist.tar.gz deqp-dist`

## Run

- Vulkan
  - `external/vulkancts/modules/vulkan/deqp-vk \
       --deqp-caselist-file=../external/vulkancts/mustpass/master/vk-default.txt \
       --deqp-log-images=disable \
       --deqp-log-shader-sources=disable`
  - `--deqp-log-filename=<file.qpa>` to specify an alternative name
  - ANV CI also has
    - `ANV_ABORT_ON_DEVICE_LOSS=true`
    - `MESA_VK_WSI_PRESENT_MODE=immediate`
    - `--deqp-surface-type=fbo`
    - `--deqp-shadercache=disable`
    - see
      <https://gitlab.freedesktop.org/Mesa_CI/mesa_jenkins/-/blob/master/vulkancts-test/build.py>
    - or <https://mesa-ci.01.org/>
      - Job: vulkancts
      - Revisions: to see the commits
      - Platforms: pick desired platform such as gen9
      - Group: dEQP-VK
- EGL/GLES
  - set `DEQP_TARGET` to one of the desired targets under `targets/`
    - e.g., `x11_egl`
  - force disable GLESv1
    - edit `targets/x11_egl/x11_egl.cmake`
    - I have to do this because my cross-compile sysroot has GLESv1 but my
      target machine does not
  - Run GLES
    - `EGL_PLATFORM=surfaceless
       external/modules/gles2/deqp-gles2 \
         --deqp-surface-type=pbuffer \
         --deqp-gl-config-name=rgba8888d24s8ms0 \
         --deqp-surface-width=256 \
         --deqp-surface-height=256`
    - all arguments required becaues of surfaceless?
    - replace gles2 by gles3 and gles31

## Android

- Build
  - `git remote add aosp https://android.googlesource.com/platform/external/deqp`
  - `git fetch aosp`
  - `git checkout -t aosp/android11-tests-dev`
    - Android 11 CTS cut releases from `android11-tests-dev`
  - `python external/fetch_sources.py`
  - `python scripts/android/build_apk.py --abis x86_64 --sdk ~/android-sdk/ --ndk ~/android-sdk/ndk/23.1.7779620/`
    - this checks for `aapt`, `zipalign`, and `dx`, where `dx` is removed
      after `build-tools;29.0.3`
  - `python scripts/android/install_apk.py`
    - might need to disable
      `Settings -> System -> Developer options -> Verify apps over ADB`
- Run
  - `adb shell am start -n com.drawelements.deqp/android.app.NativeActivity -e cmdLine '"deqp
    --deqp-case=dEQP-VK.api.object_management.multithreaded_shared_resources.device_group
    --deqp-log-filename=/sdcard/dEQP-Log.qpa"'`
- mustpass
  - android mustpass is under `android/cts/master`
  - `scripts/build_android_mustpass.py` generates the mustpass files
    - it imports `scripts/mustpass.py` which is used by android and
      non-android
    - `android/cts/master/mustpass.xml` is not used by android cts
  - `android/cts/AndroidTest.xml` is what android cts uses

## `TestResults.qpa`

- browse to `scripts/qpa_image_viewer.html` and open the qpa file
- or, <https://android.googlesource.com/platform/external/cherry/+/master>
  - `mkdir data`
  - `python ../VK-GL-CTS/scripts/build_caselists.py data`
    - or manually `./deqp-vk --deqp-runmode=xml-caselist` and copy
      `dEQP-VK-cases.xml` to `data`
  - `GO111MODULE=off go run server.go`
  - browse to `http://127.0.0.1:8080`

## Build System

- CMakeLists.txt `add_subdirectory`
  - `frameworks/delib/debase`
    - drawElements Base Portability Library
  - `frameworks/delib/decpp`
    - drawElements C++ Base Library
  - `frameworks/delib/depool`
    - drawElements Memory Pool Library
  - `frameworks/delib/dethread`
    - drawElements Thread Library
  - `frameworks/delib/destream`
    - drawElements I/O Stream Library
  - `execserver`
    - dEQP Execution Server
  - `executor`
    - dEQP Test Executor
  - `framework`
  - `external/vulkancts/framework/vulkan`
  - `modules`
  - `external/vulkancts/modules/vulkan`
  - `external/openglcts`
- `DEQP_TARGET=surfaceless`
  - `targets/surfaceless/surfaceless.cmake`
    - requires EGL/GLES2/GLES3 headers/libraries
  - `framework/platform/surfaceless`
    - `createPlatform` creates `tcu::surfaceless::Platform` which registers
      `ContextFactory`
    - no `getEGLPlatform` and thus no `deqp-egl` support
    - `getGLPlatform` is supported through `glu`
      - `ContextFactory::createContext` calls `eglGetDisplay(NULL)`
    - `getVulkanPlatform` is supported
      - it dlopens
- `DEQP_TARGET=x11_egl`
  - `targets/x11_egl/x11_egl.cmake`
    - does not require EGL/GLES2/GLES3 headers/libraries
  - `framework/platform/lnx`
    - `createPlatform` creates `tcu::lnx::LinuxPlatform` which registers
      `eglu::GLContextFactory`
    - `getEGLPlatform` is supported through `eglu`
    - `getGLPlatform` is supported through `glu`
      - `GLContextFactory::createContext` creates a `RenderContext` where
        calls `eglGetPlatformDisplay` or `eglGetDisplay`
    - `getVulkanPlatform` is supported
- `add_deqp_module` is a macro that adds a module
  - if `DE_OS_IS_ANDROID`, it skips the executable
    - we can comment this out, edit `scripts/android/build_apk.py` to build
      `deqp-vk`, and define `createPlatform` with a dummy `NativeActivity` in
      `framework/platform/android/tcuAndroidPlatform.cpp`

## Vulkan Test Initialization

- `main` is defined by `framework/platform/tcuMain.cpp`
- `tcu::App` has a `TestSessionExecutor` to manage the test session
- `tcu::TestSessionExecutor::iterate`
  - `enterTestPackage`
    - initializes Vulkan
    - initializes test package
  - `enterTestGroup`
    - picks a test group such as `dEQP-VK.info`
  - `enterTestCase`
    - picks a test case such as `dEQP-VK.info.build`
  - `iterateTestCase`
- from `tcu::TestSessionExecutor::enterTestPackage`
  - `vkt::BaseTestPackage::createExecutor`
  - `vkt::TestCaseExecutor::TestCaseExecutor`
  - `vkt::createLibrary`
  - `tcu::surfaceless::VulkanPlatform::createLibrary`
  - `tcu::surfaceless::VulkanLibrary::VulkanLibrary`
    - this is where `libvulkan.so` is loaded
- `external/vulkancts/modules/vulkan/vktTestPackage.cpp`
  - `enterTestPackage` calls `vkt::TestPackage::init` to initialize the test
    package and builds the test case hierarchy
  - `enterTestCase` calls `vkt::TestCaseExecutor::init` to initialize a test
    case
  - `iterateTestCase` calls `vkt::TestCaseExecutor::iterate` to execute a test
    case
- `external/vulkancts/modules/vulkan/api/vktApiTests.cpp`
  - `createApiTests` adds the test groups

## EGL/GLES Test Initialization

- from `tcu::DefaultHierarchyInflater::enterTestPackage`
  - `deqp::gles2::TestPackage::init`
  - `deqp::gles2::Context::Context`
  - `glu::createDefaultRenderContext`
  - `glu::createRenderContext`
  - `tcu::surfaceless::ContextFactory::createContext`
  - `tcu::surfaceless::EglRenderContext::EglRenderContext`
  - `eglw::DefaultLibrary::DefaultLibrary`
    - this is where `libEGL.so` is loaded
    - and where `eglGetProcAddress` is called the first time

## Test Case: `dEQP-VK.subgroups.ballot_broadcast.framebuffer.*`

- `external/vulkancts/modules/vulkan/subgroups/vktSubgroupsBallotBroadcastTests.cpp`
- `createSubgroupsBallotBroadcastTests`
  - for each format
    - glsl formats: int, uint, float, double, bool, and their vecs
    - for each op
      - broadcast ops: `subgroupBroadcast`, `broadcast_nonconst`
        (subgroupBroadcastDynamicId), `subgroupBroadcastFirst`
      - for each stage
        - vs, tcs, tes, gs
- `initFrameBufferPrograms`
  - `initStdFrameBufferPrograms`
    - `setFragmentShaderFrameBuffer` adds fs
      - it writes `float in_color` to `uint out_color`
    - if stage is vs,
      - see qpa for the dumped vs source code
- `noSSBOtest` -> `makeVertexFrameBufferTest`
  - `maxWidth` is hard coded to 1024
  - `extraDataCount` is 1 and `extraData[0].numElements` is hard coded to 128
  - allocates a `VkBuffer` holding 128 elements
  - `subgroupSize` is queryed from the driver
    - 128 for turnip
  - topology is `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`
  - for some widths
    - 1..128, 256, 512; total 130 iterations
    - draw `width` vertices

## Test Case: `dEQP-VK.api.copy_and_blit.core.resolve_image.diff_layout_copy_before_resolving.4_bit_general_general`

- `external/vulkancts/modules/vulkan/api/vktApiCopiesAndBlittingTests.cpp`
- `ResolveImageToImage::ResolveImageToImage`
  - it creates 3 64x64 `VK_FORMAT_R8G8B8A8_UNORM` images
    - `m_multisampledImage` has `VK_SAMPLE_COUNT_4_BIT`
    - `m_multisampledCopyImage` has `VK_SAMPLE_COUNT_4_BIT`
    - `m_destination` has `VK_SAMPLE_COUNT_1_BIT`
  - it draws a triangle
    - both `m_multisampledImage` and `m_multisampledCopyImage` are
      transitioned to `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL` by barriers
    - renderpass transitions `m_multisampledImage` to
      `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` for clearing/rendering and
      back to `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`
- `ResolveImageToImage::iterate`
  - `generateBuffer` and `uploadImage` fills out `m_destination` with
    `FILL_MODE_GRADIENT`
  - `copyMSImageToMSImage` copies `m_multisampledImage` to
    `m_multisampledCopyImage`
    - `m_multisampledImage` and `m_multisampledCopyImage` are transitioned to
      `VK_IMAGE_LAYOUT_GENERAL` by barriers
    - `vkCmdCopyImage`
  - finally
    - `m_destination` is transitioned to `VK_IMAGE_LAYOUT_GENERAL` by barriers
    - `vkCmdResolveImage`
  - `checkIntermediateCopy`
    - it creates a fb with `m_multisampledImage` and `m_multisampledCopyImage`
    - it draws a fullscreen quad, compares the results in fs, and writes
      results to an ssbo
    - it verifies the ssbo
- `dEQP-VK.api.copy_and_blit.core.resolve_image.layer_copy_before_resolving.4_bit`
  - this is similar except it uses xfer layouts and `COPY_MS_IMAGE_LAYER_TO_MS_IMAGE`
  - 1st cmdbuffer
    - both `m_multisampledImage` and `m_multisampledCopyImage` are
      transitioned to `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL` by barriers
    - clear `m_multisampledImage` to black and clear `m_multisampledCopyImage`
      to white
    - draw a triangle to layer 2 of `m_multisampledImage`
  - 2nd cmdbuffer
    - fills in `m_destination` with `FILL_MODE_GRADIENT`
    - transition `m_destination` to `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`
  - 3rd cmdbuffer
    - transition `m_multisampledImage` to
      `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL` by barrier
    - `vkCmdCopyImage` from layer 2 of `m_multisampledImage` to layer 4 of
      `m_multisampledCopyImage`
    - transition `m_multisampledCopyImage` to `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`
  - 4th cmdbuffer
    - `vkCmdResolveImage` all 5 layers from `m_multisampledCopyImage` to
      `m_destination`
  - 5th cmdbuffer
    - transition `m_destination` to `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`
      with a barrier
    - `vkCmdCopyImageToBuffer` all 5 layers from `m_destination` to a buffer
      for readback
    - transition `m_destination` to `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`
      with a barrier
  - 6th cmdbuffer
    - draw a quad to verify `m_multisampledCopyImage`
