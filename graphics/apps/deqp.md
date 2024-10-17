dEQP
====

## Build

- Build
  - `python3 external/fetch_sources.py`
  - `cmake -S . -B out -G Ninja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_COMPILER_LAUNCHER=ccache -DDEQP_TARGET=surfaceless`
    - `DEQP_TARGET` is mostly for GL/GLES
  - `ninja`
- Package
  - `cd out`
  - `mkdir deqp-dist`
  - `cd deqp-dist`
  - `for i in vulkan/deqp-vk vulkan/vulkan; do ln -sf ../external/vulkancts/modules/$i; done`
  - `for i in egl/deqp-egl gles2/deqp-gles2 gles2/gles2 gles3/deqp-gles3 gles3/gles3 gles31/deqp-gles31 gles31/gles31; do ln -sf ../modules/$i; done`
  - `for i in glcts gl_cts; do ln -sf ../external/openglcts/modules/$i; done`
  - `for i in vk-default.txt vk-default ; do ln -sf ../../external/vulkancts/mustpass/main/$i; done`
    - the other mustpasses are already under `gl_cts/data/mustpass`
  - `strip deqp-* glcts`
  - `cd ..`
  - `tar zchf deqp-dist.tar.gz deqp-dist`

## Mustpass

- `python scripts/build_caselists.py out`
  - it builds `/tmp/deqp-caselists` with `-DCMAKE_BUILD_TYPE=Debug` and
    `-DDEQP_TARGET=null`
  - it runs `deqp-{vk,egl,gles2,gles3,gles31}` with
    `--deqp-runmode=xml-caselist` to generate the caselists
  - at `vulkan-cts-1.3.9.2`,
    - `deqp-egl` has 4111 `dEQP-EGL.*` tests from `modules/egl`
    - `deqp-gles2` has 19825 `dEQP-GLES2.*` tests from `modules/gles2`
    - `deqp-gles3` has 48298 `dEQP-GLES3.*` tests from `modules/gles3`
    - `deqp-gles31` has 37942 `dEQP-GLES31.*` tests from `modules/gles31`
    - `deqp-vk` has 2728883 `dEQP-VK.*` tests from `modules/vulkan`
    - `deqp-vksc` has 886129 `dEQP-VKSC.*` tests from `modules/vulkan`
    - `glcts` has 237213 tests
      - 4111  `dEQP-EGL.*` from `modules/egl`
      - 19825 `dEQP-GLES2.*` from `modules/gles2`
      - 48298 `dEQP-GLES3.*` from `modules/gles3`
      - 37942 `dEQP-GLES31.*` from `modules/gles31`
      - 15    `CTS-Configs.*`
      - 871   `KHR-GL30.*`
      - 880   `KHR-GL31.*`
      - 1171  `KHR-GL32.*`
      - 5176  `KHR-GL33.*`
      - 6085  `KHR-GL40.*`
      - 6106  `KHR-GL41.*`
      - 24    `KHR-GL42.*`
      - 14    `KHR-GL42-COMPAT.*`
      - 6217  `KHR-GL42.*`
      - 8761  `KHR-GL43.*`
      - 9437  `KHR-GL44.*`
      - 10068 `KHR-GL45.*`
      - 10068 `KHR-GL46.*`
      - 36    `KHR-NoContext.*`
      - 79    `KHR-Single-GL43.*`
      - 145   `KHR-Single-GL44.*`
      - 6198  `KHR-Single-GL45.*`
      - 6198  `KHR-Single-GL46.*`
      - 1325  `dEQP-GL45-ES3.*`
      - 31309 `dEQP-GL45-ES31.*`
      - 473   `KHR-GLES2.*`
      - 4105  `KHR-GLES3.*`
      - 3499  `KHR-GLES31.*`
      - 1324  `KHR-GLES32.*`
      - 1400  `KHR-GLESEXT.*`
      - 6053  `KHR-Single-GLES32.*`
- `external/vulkancts/scripts/build_mustpass.py`
  - the `default` configuration
    - includes `main.txt`
    - excludes `test-issues.txt`, `excluded-tests.txt`, `android-tests.txt`
    - splits tests into groups
    - generates `vk-default.txt`
      - the naming is `<module-shortname>-<configuration>.txt`, where `vk`
        is the shortname for `dEQP-VK` module
- `external/openglcts/scripts/build_mustpass.py`
  - GL
    - `GL_CTS_KHR_MP_PROJECT`
      - `KHR-GL{30,31,32,33,40,41,42,43,44,45,46}`
      - `GTF-GL{30,31,32,33,40,41,42,43,44,45,46}`
        - for tests written in GTF (GL test framework) rather than dEQP
          framework
    - `GL_CTS_NOCTX_PROJECT`
      - `KHR-NOCTX-GL{30,40,43,45}`
    - `GL_CTS_KHR_SINGLE_PROJECT`
      - `KHR-Single-GL{43,44,45,46}`
    - `GL_CTS_GLES_PROJECT`
      - `dEQP-GL45-ES{3,31}`
  - EGL/GLES
    - `CTS_KHR_MP_ES_PROJECT`
      - `KHR-GLES{2,3,31,32,EXT}`
    - `CTS_KHR_MP_NOCTX_ES_PROJECT`
      - `KHR-NOCTX-ES{2,32}`
    - `CTS_KHR_MP_SINGLE_ES_PROJECT`
      - `KHR-Single-GLES32`
    - `CTS_AOSP_MP_ES_PROJECT`
      - `dEQP-GLES{2,3,31}`
    - `CTS_AOSP_MP_EGL_PROJECT`
      - `dEQP-EGL`
- `scripts/build_android_mustpass.py`

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

## Mustpass

- `external/vulkancts/scripts/build_mustpass.py`
  - it calls `genMustpassLists` with `MUSTPASS_LISTS`
    - `PROJECT` has `external/vulkancts/mustpass` as the path
    - there are `VULKAN_MAIN_PKG` and `VULKAN_SC_MAIN_PKG` packages
  - `VULKAN_MAIN_PKG` package has two configurations
    - `default` configuration generates `vk-default.txt`
      - it includes tests defined in `main.txt` and excludes tests defined in
        `test-issues.txt`, `excluded-tests.txt`, `android-tests.txt`
      - it splits up `dEQP-VK.*`, `dEQP-VK.pipeline.*`, `dEQP-VK.image.*`,
        `dEQP-VK.shader_object.*`
    - `fraction-mandatory-tests` configuration generates
      `vk-fraction-mandatory-tests.txt`
      - it includes tests defined in `fraction-mandatory-tests.txt`
  - what happens is
    - it builds `deqp-vk` and `deqp-vksc` in `/tmp/deqp-caselists/null-Debug`
    - it invokes them with `--deqp-runmode=txt-caselist` to generate the
      caselists
    - it applies the configurations to generate the mustpass lists
- `external/openglcts/scripts/build_mustpass.py`
  - it calls `genMustpassLists` with `ES_MUSTPASS_LISTS`
    - projects have these as the paths
      - `external/openglcts/data/mustpass/gles/khronos_mustpass`
      - `external/openglcts/data/mustpass/gles/khronos_mustpass_noctx`
      - `external/openglcts/data/mustpass/gles/khronos_mustpass_single`
      - `external/openglcts/data/mustpass/egl/aosp_mustpass`
      - `external/openglcts/data/mustpass/gles/aosp_mustpass`
    - there are many packages
  - it also calls `genMustpassLists` with `gl_mustpass_lists`
    - projects have these as the paths
      - `external/openglcts/data/mustpass/gl/khronos_mustpass`
      - `external/openglcts/data/mustpass/gl/khronos_mustpass_noctx`
      - `external/openglcts/data/mustpass/gl/khronos_mustpass_single`
      - `external/openglcts/data/mustpass/gl/aosp_mustpass`
- `scripts/build_android_mustpass.py`
  - it calls `genMustpassLists` with `MUSTPASS_LISTS`
    - `PROJECT` has `android/cts` as the path
    - there are `MAIN_EGL_PKG`, `MAIN_GLES2_PKG`, `MAIN_GLES3_PKG`,
      `MAIN_GLES31_PKG`, and `MAIN_VULKAN_PKG` packages

## Android

- Build
  - `git remote add aosp https://android.googlesource.com/platform/external/deqp`
  - `git fetch aosp`
  - `git checkout -t aosp/android11-tests-dev`
    - Android 11 CTS cut releases from `android11-tests-dev`
  - `python external/fetch_sources.py`
  - `python scripts/android/build_apk.py --abis x86_64 --sdk ~/android/sdk --ndk ~/android/sdk/ndk/26.1.10909125`
    - this checks for `aapt`, `zipalign`, `apksigner`, and `d8`/`dx`
    - older deqp only checks for `dx` which is removed after
      `build-tools;29.0.3`
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
- branches
  - `pie-cts-dev` is based on `vulkan-cts-1.1.0`
  - `android11-tests-dev` is based on `vulkan-cts-1.2.1`
  - `android13-tests-dev` is based on `vulkan-cts-1.3.1`

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

## Test Case: `dEQP-VK.renderpass2.depth_stencil_resolve.image_2d_49_13.samples_2.d16_unorm_s8_uint.depth_none_stencil_zero_testing_stencil`

- `DepthStencilResolveTest::createRenderPass`
  - two attachments
    - first
      - `VK_FORMAT_D24_UNORM_S8_UINT`
      - `VK_SAMPLE_COUNT_2_BIT`
      - `VK_ATTACHMENT_LOAD_OP_CLEAR`
      - `VK_ATTACHMENT_STORE_OP_DONT_CARE`
      - final layout `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`
    - second
      - `VK_FORMAT_D24_UNORM_S8_UINT`
      - `VK_SAMPLE_COUNT_1_BIT`
      - `VK_ATTACHMENT_LOAD_OP_CLEAR`
      - `VK_ATTACHMENT_STORE_OP_STORE`
      - final layout `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`
  - 1 subpass
    - `pDepthStencilAttachment` is att 0 with
      `VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL`
    - `pDepthStencilResolveAttachment` is att 1 with
      `VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL`
      - `depthResolveMode` is `VK_RESOLVE_MODE_NONE`
      - `stencilResolveMode` is `VK_RESOLVE_MODE_SAMPLE_ZERO_BIT`
- `DepthStencilResolveTest::createRenderPipeline`
  - `quad-vert` uses `gl_VertexIndex` to set `gl_Position` to the corners
  - `quad-frag` does
    - `if(gl_SampleID != pushConstants.sampleID) discard;`
    - `gl_FragDepth = 0.5;`
  - viewport is 49x13
  - scissor is the quad
    - offset `(10, 5)`
    - extent `(20, 8)`
  - stencil test always writes `1`
    - `VK_STENCIL_OP_REPLACE` and `VK_COMPARE_OP_ALWAYS`
- draw
  - a barrier from `VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT` to
    `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`
  - begin a render pass for the entire 49x13 area
    - clear value is `(1.0, 5)`
  - for each sample,
    - bind the pipeline
    - push the sample id as push const
    - set the stencil ref value
      - first sample is 1
      - second sample is 255
    - draw with a rectangle from `(10, 5)` to `(20, 8)`
  - end the render pass
- copy image to buffer
  - a barrier to make `m_singlesampleImage`'s
    `VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT` available to
    `VK_ACCESS_TRANSFER_READ_BIT`
  - copy the entire 49x13 `m_singlesampleImage` to `m_buffer`
  - a barrier to make `m_buffer`'s `VK_ACCESS_TRANSFER_WRITE_BIT` available to
    `VK_ACCESS_HOST_READ_BIT`
- verify stencil
  - for each pixel,
    - if outside of the draw rectangle, it should be the clear value `5`
    - if inside of the draw rectangle, it should be `1`
- `dEQP-VK.renderpass2.depth_stencil_resolve.image_2d_16_64_6.samples_2.d16_unorm.depth_zero` is similar
  - image size `16x64`
  - `imageLayers` is 6, `viewLayers` is 3, and `resolveBaseLayer` is 0
  - `renderArea` has offset `(10, 10)` and extent `(6, 54)`
  - `depthResolveMode` is `VK_RESOLVE_MODE_SAMPLE_ZERO_BIT`
  - `verifyBuffer` is `VB_DEPTH`
  - `clearValue.depth` is `1.0`
  - `depthExpectedValue` is `0.04`
  - `quad-geom` is used to draw to first 3 layers
  - `quad-frag` writes `gl_FragDepth`
    - sample 0 uses `0.04` and sample 1 uses `0.02`
  - `DepthStencilResolveTest::verifyDepth` verifies that, for each pixel in
    the first 3 layers,
    - pixels outside of the render area are all `clearValue.depth`
    - pixels inside the render area are all `depthExpectedValue`

## Test Case: `dEQP-VK.pipeline.monolithic.timestamp.calibrated.*`

- properties and features
  - `VkQueueFamilyProperties::timestampValidBits` gives the timestamp bits of
    a queue family
  - `VkPhysicalDeviceLimits::timestampPeriod` gives the hw clock period in ns
    - if 1000ns, the hw clock freq is 1mhz
  - `vkGetPhysicalDeviceCalibrateableTimeDomainsEXT` returns the available
    domains
    - radv supports `VK_TIME_DOMAIN_DEVICE_EXT`,
      `VK_TIME_DOMAIN_CLOCK_MONOTONIC_EXT`, and
      `VK_TIME_DOMAIN_CLOCK_MONOTONIC_RAW_EXT`
- test ctor
  - `vkCmdWriteTimestamp` to write the timestamp of
    `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`
- `dev_domain_test`
  - `vkGetCalibratedTimestampsEXT` to get the before timestamp
  - submit the `vkCmdWriteTimestamp`
  - `vkGetCalibratedTimestampsEXT` to get the after timestamp
  - the 3 timestamps must be in order
- `host_domain_test`
  - `clock_gettime` to get the before timestamp
  - `vkGetCalibratedTimestampsEXT` to get the before timestamp in the same
    domain
  - `clock_gettime` to get the after timestamp
  - the 3 timestamps must be in order
- `calibration_test`
  - `vkGetCalibratedTimestampsEXT` to get the before timestamp for both the
    dev and host domains
  - sleep 200ms
  - `vkGetCalibratedTimestampsEXT` to get the after timestamp for both domains
  - in the dev domain, before and after should be ~200ms
  - in the host domain, before and after should be ~200ms

## Test Case: `dEQP-VK.glsl.builtin.precision.fract.highp.*`

- `addBuiltinPrecisionTests` adds all test cases
  - `DEFINE_DERIVED_FLOAT1(Fract, fract, x, x - app<Floor32Bit>(x))` defines
    `class Fract : public DerivedFunc`
  - `addScalarFactory<Fract>` adds a `GenFuncCaseFactory` factory
  - `createFuncGroup` adds the test cases for the factory
    - the shader type is `glu::SHADERTYPE_COMPUTE`
  - `GenFuncCaseFactory::createCase` calls `createFuncCase` to create
    `FuncCase`
- `FuncCase::buildTest` calls `PrecisionCase::testStatement` to initialize
  `ShaderSpec`
  - the statement is `out0 = fract(in0);`
- `FuncCase::createInstance` calls `BuiltinPrecisionCaseTestInstance` to
  create a `TestInstance`
  - `vkt::shaderexecutor::createExecutor` creates a `ComputeShaderExecutor`
- `PrecisionCase::initPrograms` generates the GLSL compute code according to
  `ShaderSpec`

    precision highp float;
    layout(local_size_x = 1) in;
    struct Inputs { highp float in0; };
    struct Outputs { highp float out0; };
    layout(set = 0, binding = 0, std430) buffer InBuffer { Inputs inputs[]; };
    layout(set = 0, binding = 1, std430) buffer OutBuffer { Outputs outputs[]; };
    void main (void)
    {
            uint invocationNdx = gl_NumWorkGroups.x*gl_NumWorkGroups.y*gl_WorkGroupID.z
                               + gl_NumWorkGroups.x*gl_WorkGroupID.y + gl_WorkGroupID.x;
            float in0 = float(inputs[invocationNdx].in0);
            float out0;
            out0 = fract(in0);
            outputs[invocationNdx].out0 = out0;
    }
- `BuiltinPrecisionCaseTestInstance::iterate` executes the test
  - it prepares inputs and outputs
  - it calls `ShaderExecutor::execute` to execute the test on the gpu
  - it calls `Statement::execute` to execute the test on the cpu
    - that in turn calls `Expr::evaluate`
    - the expression is `x - app<Floor32Bit>(x)`
    - `operator-` applies `Sub` on the two operands
    - `Sub::doApply` calls `TCU_SET_INTERVAL_BOUNDS` to calculate the interval
      - I am seeing bugs on some versions of clang with `-O2`

## Test Case: `dEQP-VK.api.external.fence.sync_fd.export_multiple_times_temporary`

- `testFenceMultipleExports`
  - `vkCreateDevice`
  - `vkCreateFence` with `VK_EXTERNAL_FENCE_HANDLE_TYPE_OPAQUE_FD_BIT`
- it then loops for `exportCount` (1024) times
  - `submitAtomicCalculationsAndGetFenceNative` does everything
    - create a cmd pool, cmd buf, event, buffer, shader, pipeline, descriptor,
      etc.
  - `tuneWorkSizeYAndPrepareCommandBuffer` records a compute dispatch with the
    right workgroup count
    - it picks a workgroup count that takes the gpu 9ms to execute
    - it then records the same command again
  - submit and get the sync fd
  - wait for queue idle

## Test Case: `dEQP-VK.ycbcr.plane_view.memory_alias.*`

- `populateViewTypeGroup` loops through all ycbcr formats
  - size is `32x58`
  - only multi-planar ycbcr formats are considered
  - image flags are
    - `VK_IMAGE_CREATE_MUTABLE_FORMAT_BIT`
    - `VK_IMAGE_CREATE_ALIAS_BIT`
    - `VK_IMAGE_CREATE_DISJOINT_BIT`
  - `addPlaneViewCase` is called for each plane of each ycbcr format
    - take `VK_FORMAT_G8_B8_R8_3PLANE_420_UNORM` for example, it creates
      - `g8_b8_r8_3plane_420_unorm_plane_0`
      - `g8_b8_r8_3plane_420_unorm_plane_1`
      - `g8_b8_r8_3plane_420_unorm_plane_2`
    - it also creates a test for each compatible formats such as
      - `g8_b8_r8_3plane_420_unorm_plane_0_compatible_format_r4g4_unorm_pack8`
      - `g8_b8_r8_3plane_420_unorm_plane_0_compatible_format_r8_uint`
      - `g8_b8_r8_3plane_420_unorm_plane_0_compatible_format_r8_sint`
- `getShaderSpec` returns a fs spec that looks like

    #version 450
    layout(binding = 1, set = 1) uniform highp sampler2D u_image;
    layout(binding = 0, set = 1) uniform highp sampler2D u_planeView;
    layout(location=0) flat in highp vec2 vtx_out_texCoord;
    layout(location=0) out highp vec4 o_result0;
    layout(location=1) out highp vec4 o_result1;
    void main (void) {
            highp vec2 texCoord = vtx_out_texCoord;
            highp vec4 result0;
            highp vec4 result1;
            result0 = texture(u_image, texCoord);
            result1 = vec4(texture(u_planeView, texCoord));
            o_result0 = result0;
            o_result1 = result1;
    }
- `testPlaneView` setup
  - creates two images
    - `image` is the entire ycbcr disjoint image
    - `imageAlias` is a plane image with compatible format
    - `allocateAndBindImageMemory` allocates and bind memories for each plane of
      `image`
    - `imageAlias` aliases one of the memories
  - creates two image views
    - `wholeView` is a `VK_IMAGE_ASPECT_COLOR_BIT` view of `image` with ycbcr conversion
    - `planeView` is a `VK_IMAGE_ASPECT_COLOR_BIT` view of `imageAlias`
  - creates two samplers
    - `wholeSampler` is a sampler for `wholeView` with ycbcr conversion
    - `planeSampler` is a sampler for `planeView`
  - `descLayout`
    - binding 0 is a combined sampler
    - binding 1 is an immutable combined sampler of `wholeSampler`
  - `descSet`
    - binding 0 is `planeViewSampler` and `planeView`
    - binding 1 is `wholeViewSampler` and `wholeView`
  - `imageData` contains random data
  - `imageAlias` is transitioned to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`
  - `uploadImage` initializes `image`
    - it transitions `image` to `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`
    - `vkcmdCopyBufferToImage` to copy `imageData` to `image`
    - another transition to `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`
- `testPlaneView` run
  - it picks random 500 points
  - it executes the shader to get the values at those 500 points
  - it computes reference values in `referenceWhole` and `referencePlane`
  - it compares the values against the reference values

## Test Case: `dEQP-VK.api.copy_and_blit.core.image_to_image.3d_images.2d_to_3d_whole`

- `addImageToImage3dImagesTests`
  - `params2DTo3D.src.image` is 2d, r8g8b8a8 uint, 32x32x16
  - `params2DTo3D.dst.image` is 3d, r8g8b8a8 uint, 32x32x16
  - `params2DTo3D.regions` has extent 32x32x16, with layer count 16 for 2d and
    layer count 1 for 3d
- `CopyImageToImage::CopyImageToImage`
  - `m_source` is the src image, 32x32x1 and 16 layers
  - `m_destination` is the dst image, 32x32x16 and 1 layer
- `CopyImageToImage::iterate`
  - `m_sourceTextureLevel` and `m_destinationTextureLevel` are
    `tcu::TextureLevel` for src and dst
    - `tcu::TextureLevel` holds the pixel data of a single level in a
      `deAlignedMalloc`ed buffer
    - `tcu::TextureLevel::getAccess` returns a `tcu::PixelBufferAccess` for
      the buffer
  - `CopiesAndBlittingTestInstance::generateBuffer` fills in the pixel data
    with `FILL_MODE_GRADIENT` pattern
  - `CopiesAndBlittingTestInstance::generateExpectedResult` calls
    `CopyImageToImage::copyRegionToTextureLevel` to emulate gpu copy
  - `CopiesAndBlittingTestInstance::uploadImage` allocates a vk buffer, copies
    the pixel data into vk buffer, and uses `vkCmdCopyBufferToImage` to
    initialize the vk image
  - it uses `vkCmdCopyImage` to copy from the src 2d image to the dst 3d image
  - `CopiesAndBlittingTestInstance::readImage` reads from the dst 3d image to
    a cpu buffer
- `CopyImageToImage::checkTestResult` checks that the final pixel data written
  by the gpu copy match those emulated by cpu copy

## Test Case: `dEQP-VK.spirv_assembly.instruction.compute.convertutof.uint64_to_float32`

- `createInstructionTests` calls `createConvertComputeTests`
  - `instruction` is `OpConvertUToF`
  - `name` is `convertutof`
  - `createConvertCases` creates a bunch of `ConvertCase`, which describes
    test input/output/assembly/type/etc.
  - `getVulkanFeaturesAndExtensions` checks features and exts
  - a test of class `SpvAsmComputeShaderCase` is created

## Test Case: `dEQP-GLES31.functional.vertex_attribute_binding.usage.single_binding.unaligned_offset_elements_1_aligned_elements`

- the test is `SingleBindingCase` with `FLAG_BUF_UNALIGNED_OFFSET` and
  `FLAG_ATTRIB_ALIGNED`
  - this is the only test with `FLAG_BUF_UNALIGNED_OFFSET`
  - `m_unalignedData` is false
  - `spec.bufferOffset` is 19
    - this is the offset of the first usable byte and is intentionally
      unaligned
  - `spec.bufferStride` is 20
    - this is the size of the vertex data of a vertex
  - `spec.positionAttrOffset` is 1
    - this is the offset from `spec.bufferOffset` to valid vertex data, to
      keep vertex data aligned
  - `spec.colorAttrOffset` is -1
  - `spec.hasColorAttr` is false
- `BindingRenderCase::init`
  - `m_vao` is from `glGenVertexArrays`
  - `m_buffer` is from `SingleBindingCase::createBuffers`
    - there are `GRID_SIZE`x`GRID_SIZE` (20x20) squares
    - each square takes 2 triangles, or 6 vertices,
    - vertex data starts at `m_spec.bufferOffset + m_spec.positionAttrOffset`
      and each takes up `m_spec.bufferStride`
  - `m_program` is from `SingleBindingCase::createShader`

    in highp vec4 a_position;
    uniform highp vec4 u_color;
    out highp vec4 v_color;
    void main (void)
    {
            gl_Position = a_position;
            v_color = u_color;
    }
    
    in mediump vec4 v_color;
    layout(location = 0) out mediump vec4 fragColor;
    void main (void)
    {
            fragColor = v_color;
    }
  - `m_program` is from `SingleBindingCase::createShader`
- `BindingRenderCase::iterate`
  - it uses a `TEST_RENDER_SIZE`x`TEST_RENDER_SIZE` (64x64) surface
    - a `tcu::Surface` is an image with storage in cpu memory
  - `SingleBindingCase::renderTo` is straightforward
    - it clears to black
    - `glBindVertexBuffer` with `m_spec.bufferOffset` and `m_spec.bufferStride`
    - `glVertexAttribFormat` with `m_spec.positionAttrOffset` as the relative
      offset
    - `glUniform4f` to provide the uniform color green
    - `glDrawArrays` all vertices
    - `glReadPixels` to read back
- angle
  - `VkPipelineVertexInputStateCreateInfo` with 1 buffer and 1 attr
    - buffer stride will be provided dynamically because of
      `VK_DYNAMIC_STATE_VERTEX_INPUT_BINDING_STRIDE`
    - attr offset is 1 (from `spec.positionAttrOffset`)
  - `vkCmdBindVertexBuffers2EXT`
    - offset 19 (from `spec.bufferOffset`)
    - stride 20 (from `spec.bufferStride`)
  - `vkCmdPushConstants` of size 32 bytes
  - `vkCmdBindDescriptorSets` with 1 descriptor set
    - the descriptor set has two `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC`
      of size 16 bytes
    - the uniform buffer sizes are 67584 and 2064 respectively

## Test Case: `dEQP-GLES31.functional.shaders.sample_variables.sample_mask_in.bit_count_per_two_samples.multisample_texture_4`

- `SampleMaskCountCase` with `SampleMaskCountCase::RUN_PER_TWO_SAMPLES`
- `SampleMaskCountCase::init` calls all the way up to
  `MultisampleRenderCase::init`
  - `SampleMaskCountCase::genFragmentSource` returns fs

    #extension GL_OES_sample_variables : require
    layout(location = 0) out mediump vec4 fragColor;
    uniform highp int u_minBitCount;
    uniform highp int u_maxBitCount;
    void main (void)
    {
            mediump int maskBitCount = 0;
            for (int i = 0; i < 32; ++i)
                    if (((gl_SampleMaskIn[0] >> i) & 0x01) == 0x01)
                            ++maskBitCount;
            if (maskBitCount < u_minBitCount || maskBitCount > u_maxBitCount)
                    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
            else
                    fragColor = vec4(0.0, 1.0, 0.0, 1.0);
    }
- `SampleMaskCountCase::preDraw`
  - it sets `minBitCount` to 1 and `maxBitCount` to 3
    - `(m_numTargetSamples + 1) / 2`
  - `SampleMaskBaseCase::preDraw`
    - `gl.enable(GL_SAMPLE_SHADING)`
    - `gl.minSampleShading(0.5f)`
    - this invokes the fs for "0.5" of the samples
      - in this case, the spec requires at least `max(ceil(4 * 0.5),1) = 2`
        invocations
    - for each invocation, only the active sample is set in `gl_SampleMaskIn` 
- `gl_SampleMaskIn`
  - Bit n of element w in the array is set if and only if the sample numbered
    32w + n is considered covered for this fragment shader invocation.
  - When rendering to a non-multisample buffer, or if multisample
    rasterization is disabled, all bits are zero except for bit zero of the
    first array element. That bit will be one if the pixel is covered and zero
    otherwise.
  - Bits in the sample mask corresponding to covered samples that will be
    killed due to SAMPLE_COVERAGE or SAMPLE_MASK will not be set.
  - When per-sample shading is active due to the use of a fragment input
    qualified by sample or due to the use of the gl_SampleID or
    gl_SamplePosition variables, only the bit for the current sample is set in
    gl_SampleMaskIn.
  - When state specifies multiple fragment shader invocations for a given
    fragment, the bit correspondi

## Test Case: `dEQP-GLES31.functional.blend_equation_advanced.msaa.multiply`

- `AdvancedBlendCase` and
  - `m_blendMode` is `GL_MULTIPLY`
  - `m_overdrawCount` is 1
    - that is, overdraw once and blend
  - `m_coherentBlending` is false
  - `m_rtType` is `RENDERTARGETTYPE_MSAA_FBO`
  - render with is 256x256 while viewport is 128x128
- `AdvancedBlendCase::init`
  - the shader is simple

    in highp vec4 a_position;
    in mediump vec4 a_color;
    out mediump vec4 v_color;
    void main()
    {
            gl_Position = a_position;
            v_color = a_color;
    }

    in mediump vec4 v_color;
    layout(blend_support_multiply) out;
    layout(location = 0) out mediump vec4 o_color;
    void main()
    {
            o_color = v_color;
    }
  - `m_referenceRenderer` is a `ReferenceQuadRenderer`
  - `m_refColorBuffer` is a `TextureLevel`
    - it's a cpu memory of the size of the viewport
    - `TextureFormat::RGBA` and `TextureFormat::UNORM_INT8`
  - `m_fbo` is a msaa fbo
  - `m_resolveFbo` is a resolved fbo
- `AdvancedBlendCase::iterate`
  - render with GL
    - clear and draw a quad
    - `glBlendBarrier`
    - draw a quad again to blend
  - render with cpu
  - resolve from `m_fbo` to `m_resolveFbo`
  - read pixels back and compare
  - repeat for 5 iterations
- angle
  - barrier to transition both msaa and resolve images from
    `VK_IMAGE_LAYOUT_UNDEFINED` to `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
  - render pass
    - 2 attachments
      - att0 is msaa img with initial and final layout
        `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
      - att1 is resolve img with initial and final layout
        `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
    - 1 subpass
      - 1 input att with `VK_IMAGE_LAYOUT_GENERAL` for att0
      - 1 color att with `VK_IMAGE_LAYOUT_GENERAL` for att0
      - 1 resolve att with `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` for att1
    - draw a quad
    - memory barrier to make
      `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT/VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT`
      visible to
      `VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT/VK_ACCESS_INPUT_ATTACHMENT_READ_BIT`
    - draw another quad to blend
  - barrier to transition resolve image from
    `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` to
    `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`
  - copy resolve image to buffer and compare
  - note that this repeats 5 iterations and the barriers are different since
    the second iteration
    - the pre-pass barrier transitions the msaa image from
      `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL` to
      `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
    - an additional barrer before the first draw to make
      `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT/VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT`
      visible to
      `VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT/VK_ACCESS_INPUT_ATTACHMENT_READ_BIT`

## Test Case: `dEQP-VK.image.texel_view_compatible.compute.basic.2d_image.image_store.astc_4x4_unorm_block.r32g32b32a32_uint`

- `createImageCompressionTranscodingTests` creates the test
- `TestParameters` is
  - `operation` is `OPERATION_IMAGE_STORE`
  - `shader` is `SHADER_TYPE_COMPUTE`
  - `size` is `64x64x1`
  - `layers` is 1
  - `imageType` is `IMAGE_TYPE_2D`
  - `formatCompressed` is `VK_FORMAT_ASTC_4x4_UNORM_BLOCK`
  - `formatUncompressed` is `VK_FORMAT_R32G32B32A32_UINT`
  - `imagesCount` is 3
  - `compressedImageUsage` is xfer src, xfer dst, sampled, and storage
  - `compressedImageViewUsage` is the same as `compressedImageUsage`
  - `uncompressedImageUsage` is the same as `compressedImageUsage`
  - `useMipmaps` is false
  - `formatForVerify` is `VK_FORMAT_R8G8B8A8_UNORM`
  - `formatIsASTC` is true
- `TexelViewCompatibleCase`
  - `initPrograms`
    - `comp`
      - `layout (local_size_x = 1, local_size_y = 1, local_size_z = 1) in;`
      - `ivec2 pos = ivec2(gl_GlobalInvocationID.xy);`
      - `imageStore(u_image0, pos, imageLoad(u_image1, pos));`
      - `imageStore(u_image2, pos, imageLoad(u_image0, pos));`
    - `decompress`
      - `layout (local_size_x = 1, local_size_y = 1, local_size_z = 1) in;`
      - `layout (binding = 0) uniform sampler2D compressed_result;`
      - `layout (binding = 1) uniform sampler2D compressed_reference;`
      - `layout (binding = 2, rgba8) writeonly uniform image2D decompressed_result;`
      - `layout (binding = 3, rgba8) writeonly uniform image2D decompressed_reference;`
      - `const vec2 pixels_resolution = vec2(gl_NumWorkGroups.xy);`
      - `const vec2 cord = vec2(gl_GlobalInvocationID.xy) / vec2(pixels_resolution);`
      - `const ivec2 pos = ivec2(gl_GlobalInvocationID.xy);`
      - `imageStore(decompressed_result, pos, texture(compressed_result, cord));`
      - `imageStore(decompressed_reference, pos, texture(compressed_reference, cord));`
  - `createInstance` creates a `ImageStoreComputeTestInstance`
    - `ImageStoreComputeTestInstance` inherits from `BasicComputeTestInstance`
- `BasicComputeTestInstance::iterate`
  - there are 3 images
    - image 0 has format `VK_FORMAT_ASTC_4x4_UNORM_BLOCK` and its view has
      format `VK_FORMAT_R32G32B32A32_UINT`
    - image 1 and 2 have format `VK_FORMAT_R32G32B32A32_UINT` and its view has
      the same format
  - `copyDataToImage` copies the generated compressed data into image 1
  - there are 3 descriptors
    - desc 0 has type `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`
    - desc 1 and 2 has type `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE`
  - `executeShader` executes `comp`
    - it loads raw data from image 1 and stores them to image 0
    - it then loads raw data from image 0 and stores them to image 2
  - pre-decompress verification
    - it makes sure the generated compressed data is the same as the data read
      back from image 2
  - it also calls `decompressImage`
- `BasicComputeTestInstance::decompressImage`
  - there are 5 images
    - `compressed` is from image 0 in `BasicComputeTestInstance::iterate` and
      has view `compressedView`
    - `uncompressed` is from image 2 in `BasicComputeTestInstance::iterate`
    - `resultImage` has format `VK_FORMAT_R8G8B8A8_UNORM` and has view `resultView` of the same format
    - `referenceImage` has format `VK_FORMAT_R8G8B8A8_UNORM` and has view
      `referenceView` of the same format
    - `uncompressedImage` has format `VK_FORMAT_ASTC_4x4_UNORM_BLOCK` and has
      `uncompressedView` of the same format
  - there are 4 descriptors
    - desc 0: `uncompressedView`
    - desc 1: `compressedView`
    - desc 2: `resultView`
    - desc 3: `referenceView`
  - cmd buffer
    - copy from `uncompressed` to `transferBuffer`
    - copy from `transferBuffer` to `uncompressedImage`
    - dispatch with full size (64x64x1)
      - the shader samples from `uncompressedView` and stores to `resultView`
      - the shader also samples from `compressedView` and stores to `referenceView`
    - copy from `resultImage` to `resultBuffer`
    - copy from `referenceImage` to `referenceBuffer`
  - compare `resultBuffer` and `referenceBuffer`
  - in summary,
    - in `interate`, image 0, 1, and 2 all have compressed data
      - image 0 is from load/store of image 1
      - image 1 is from copy
      - image 2 is from load/store of image 0
    - in `decompressImage`,
      - the reference is from sampling image 0 (named `compressed`) and
        storing to `referenceImage`
      - the result is from sampling image 2 (named `uncompressed`) and storing
        to `resultImage`

## Test Case: `dEQP-VK.texture.compressed_3D.astc_4x4_unorm_block_3d_pot`

- the test is a part of `create3DTextureCompressedFormatTests`
  - size is width 128, height 64, depth 8, non-mipmapped
  - format is `VK_FORMAT_ASTC_4x4_UNORM_BLOCK`
  - backing mode is regular (non-sparse)
- `Compressed3DTestInstance::Compressed3DTestInstance`
  - `TestTexture::populateCompressedLevels` initializes the 3D texture
    - with complete mipmap
    - `vkCmdCopyBufferToImage` with 8 regions, one for each level
      - level 0 is 128x64x8
      - level 1 is 64x32x4
      - level 7 is 1x1x1
- `Compressed3DTestInstance::iterate`
  - because depth is 8, it tests slice 0, 3, and 7
  - `m_renderer2D.renderQuad` draws a quad, sampling the respective slices, an
    - `vkCmdDrawIndexed` with 6 vertices
  - `vkCmdCopyImageToBuffer` to verify

## Test Case: `dEQP-VK.robustness.robustness2.bind.notemplate.r32i.unroll.nonvolatile.storage_image.fmt_qual.img.samples_1.2d.rgen`

- `createRobustness2Tests` creates the tests
  - `pushCases` is `bind` or `push`
  - `tempCases` is `notemplate` or `template`
  - `fmtCases` is `r32i`, `r32ui`, `r32f`, etc.
  - `unrollCases` is `dontunroll` or `unroll`
  - `volCases` is `nonvolatile` or `volatile`
  - `fullDescCases` is `uniform_buffer`, `storage_buffer`, `storage_image`,
    etc.
  - `fmtQualCases` is `no_fmt_qual` or `fmt_qual`
  - `fullLenCases32Bit` is `null_descriptor`, `img`, `len_4`, etc.
  - `sampCases` is `samples_1` or `samples_4`
  - `viewCases` is `1d`, `2d`, `3d`, etc.
  - `stageCases` is `comp`, `frag`, `vert`, `rgen`
  - for this specific test, `CaseDef` has
    - `format` is `VK_FORMAT_R32_SINT`
    - `stage` is `STAGE_RAYGEN`
    - `allShaderStages` is all `VK_SHADER_STAGE_*_BIT`
    - `allPipelineStages` is all `VK_PIPELINE_STAGE_*_SHADER_BIT`
    - `descriptorType` is `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE`
    - `viewType` is `VK_IMAGE_VIEW_TYPE_2D`
    - `samples` is `VK_SAMPLE_COUNT_1_BIT`
    - `bufferLen` is `0`
    - `unroll` is `true`
    - `vol` is `false`
    - `nullDescriptor` is `false`
    - `useTemplate` is `false`
    - `formatQualifier` is `true`
    - `pushDescriptor` is `false`
    - `testRobustness2` is `true`
    - `pipelineRobustnessCase` is `false`
    - `imageDim[3]` is `{5, 11, 6}`
    - `readOnly` is `false`
- `RobustnessExtsTestCase::checkSupport` checks for test requirements
- `RobustnessExtsTestCase::initPrograms` initializes the shaders

    uvec4 abs(uvec4 x) { return x; }
    int smod(int a, int b) { if (a < 0) a += b*(abs(a)/b+1); return a%b; }
    int refData[4] = {305419896, 591751049, 878082192, 1164413185};
    ivec4 zzzz = ivec4(0);
    ivec4 zzzo = ivec4(0, 0, 0, 1);
    ivec4 expectedIB;
    layout(r32i, set = 0, binding = 0) uniform iimage2D image0_0;
    layout(r32i, set = 0, binding = 1) uniform iimage2D image0_1;
    void main()
    {
      ivec4 accum = ivec4(0);
      ivec4 temp;
      int temp_ql;
      int inboundcoords, clampedLayer;
      ivec4 expectedIB2;
      [[unroll]] for (int c = -10; c <= 10; ++c) {
        int idx = smod(c * 1, 4);
        expectedIB.x = refData[0];
        expectedIB.y = 0;
        expectedIB.z = 0;
        expectedIB.w = 1;
        inboundcoords = 5;
        if (c < 0 || c >= inboundcoords) imageStore(image0_1, ivec2(c, 0), ivec4(123));
        if (c < 0 || c >= inboundcoords) imageAtomicAdd(image0_1, ivec2(c, 0), int(10));
        inboundcoords = 11;
        if (c < 0 || c >= inboundcoords) imageStore(image0_1, ivec2(0, c), ivec4(123));
        if (c < 0 || c >= inboundcoords) imageAtomicAdd(image0_1, ivec2(0, c), int(10));
        inboundcoords = 5;
        temp = imageLoad(image0_1, ivec2(c, 0));
        expectedIB2 = expectedIB;
        if (c >= 0 && c < inboundcoords) {
           if (temp == expectedIB2) temp = ivec4(0); else temp = ivec4(1);
        }
        else if (temp == zzzo) temp = ivec4(0);
        else if (temp == ivec4(123)) temp = ivec4(0);
        else temp = ivec4(1);
        accum += abs(temp);
        inboundcoords = 11;
        temp = imageLoad(image0_1, ivec2(0, c));
        expectedIB2 = expectedIB;
        if (c >= 0 && c < inboundcoords) {
           if (temp == expectedIB2) temp = ivec4(0); else temp = ivec4(1);
        }
        else if (temp == zzzo) temp = ivec4(0);
        else if (temp == ivec4(123)) temp = ivec4(0);
        else temp = ivec4(1);
        accum += abs(temp);
      }
      ivec4 color = (accum != ivec4(0)) ? ivec4(0,0,0,0) : ivec4(1,0,0,1);
      imageStore(image0_0, ivec2(gl_LaunchIDEXT.xy), color);
    }
- `RobustnessExtsTestCase::createInstance` creates a
  `RobustnessExtsTestInstance`
- `RobustnessExtsTestInstance::iterate`
  - `descriptorSetLayout` is a descriptor set layout with 2 bindings
    - binding 0 is the output image of type `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE`
    - binding 1 is the test image of type `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE`
  - `descriptorPool` is a descriptor pool
  - `descriptorSet` is a descriptor set
  - `buffer` is a buffer of size 256 and is memset to `0x3f`
  - `cmdPool` is a command pool
  - `cmdBuffer` is a command buffer
  - `images` are the output and the test images
    - the output image is always 2D of size 8x8
      - it is `vkCmdClearColorImage` to 0
      - it is transitioned to `VK_IMAGE_LAYOUT_GENERAL` /
        `VK_ACCESS_SHADER_READ_BIT`
    - the test image is specified by `CaseDef`, 2D of size 5x11 in this test
      case
      - it is `vkCmdClearColorImage` to `refData`, which is `0x12345678`
      - it is transitioned to `VK_IMAGE_LAYOUT_GENERAL` /
        `VK_ACCESS_SHADER_READ_BIT`
  - `imageViews` are the output and the test image views
  - `sampler` is a sampler
  - `pipelineLayout` is a pipeline layout
  - `copyBuffer` is for readback of `images[0]`
  - `descriptorSet` is updated and bound
    - the two bindings are updated to the two images in `images`
  - `sgHandleSize` is set to
    `VkPhysicalDeviceRayTracingPipelinePropertiesKHR::shaderGroupHandleSize`
    - it is 32 on radv
  - `pipeline` is created with `vkCreateRayTracingPipelinesKHR`
    - the only shader has stage `VK_SHADER_STAGE_RAYGEN_BIT_KHR`
    - there is only one `VkRayTracingShaderGroupCreateInfoKHR`
  - `sbtBuffer` is of size `sgHandleSize`
  - `vkGetRayTracingShaderGroupHandlesKHR` writes the opaque group handle into
    `sbtBuffer`
  - `sbtAddress` is the device address of `sbtBuffer`
  - `rgenSBTRegion` is initialized to `sbtAddress` and `sgHandleSize`
  - `images[0]` is transitioned from undefined to general
    - the initialization above becomes pointless..
  - `pipeline` is bound
  - `images[0]` is `vkCmdClearColorImage` to 0 again
  - `vkCmdPipelineBarrier` is emitted for shader read/write
  - `vkCmdTraceRaysKHR` is called with dim of the output image (8x8)
  - `vkCmdPipelineBarrier` is emitted for xfer read/write
  - `vkCmdCopyImageToBuffer` copies from `images[0]` to `copyBuffer`
  - `vkCmdPipelineBarrier` is emitted for host read
  - `vkQueueSubmit` and `vkWaitForFences`
  - `copyBuffer` is validated (all values should be 1)

## Test Case: `dEQP-VK.fragment_shading_rate.renderpass2.monolithic.fragdepth_baselevel.dynamic.attachment.noshaderrate.keep.replace.4x4.samples1.vs`

- `FragmentShadingRate::createTests` calls
  `createPipelineConstructionTypePermutations` to create the tests
  - `FragmentShadingRate::SharedGroupParams`
    - `useDynamicRendering` is false
    - `useSecondaryCmdBuffer` is false
    - `secondaryCmdBufferCompletelyContainsDynamicRenderpass` is false
    - `pipelineConstructionType` is `PIPELINE_CONSTRUCTION_TYPE_MONOLITHIC`
      - that is, `monolithic`
  - `createTests` calls `createBasicTests`
    - `groupCases` is `basic`, `fragdepth`, `fragstencil`,
      `fragdepth_baselevel`, etc.
    - `dynCases` is `dynamic` or `static`
    - `attCases` is `noattachment`, `attachment`, etc.
    - `shdCases` is `noshaderrate` or `shaderrate`
    - `combCases` is `keep`, `replace`, `min`, `max`, or `mul`
    - `extentCases` is `1x1`, `4x4`, `35x35`, etc.
    - `sampCases` is `samples1`, `samples2`, etc.
    - `shaderCases` is `vs`, `gs`, or `ms`
  - for this specific test, `CaseDef` is
    - `groupParams` is from `FragmentShadingRate::SharedGroupParams`
    - `seed` is a sequential number
    - `framebufferDim` is `4x4`
    - `samples` is `VK_SAMPLE_COUNT_1_BIT`
    - `combinerOp[2]` is `{ VK_FRAGMENT_SHADING_RATE_COMBINER_OP_KEEP_KHR, VK_FRAGMENT_SHADING_RATE_COMBINER_OP_REPLACE_KHR }`
    - `attachmentUsage` is `AttachmentUsage::WITH_ATTACHMENT`
    - `shaderWritesRate` is false
    - `geometryShader` is false
    - `meshShader` is false
    - `useDynamicState` is `true`
    - `useApiSampleMask` is false
    - `useSampleMaskIn` is false
    - `conservativeEnable` is false
    - `conservativeMode` is `VK_CONSERVATIVE_RASTERIZATION_MODE_OVERESTIMATE_EXT`
    - `useDepthStencil` is `true`
    - `fragDepth` is `true`
    - `fragStencil` is false
    - `multiViewport` is false
    - `colorLayered` is false
    - `srLayered` is false
    - `numColorLayers` is 1
    - `multiView` is false
    - `correlationMask` is false
    - `interlock` is false
    - `sampleLocations` is false
    - `sampleShadingEnable` is false
    - `sampleShadingInput` is false
    - `sampleMaskTest` is false
    - `earlyAndLateTest` is false
    - `garbageAttachment` is false
    - `dsClearOp` is false
    - `dsBaseMipLevel` is `1`
    - `multiSubpasses` is false
    - `maintenance6` is false
- `FSRTestCase::checkSupport` checks for support
- `FSRTestCase::initPrograms` initializes shaders
  - `comp`

    layout(set = 0, binding = 1) uniform utexture2DArray colorTex;
    layout(set = 0, binding = 2, std430) buffer Block0 { uvec4 b[]; } colorbuf;
    layout(set = 0, binding = 4, std430) buffer Block1 { float b[]; } depthbuf;
    layout(set = 0, binding = 5, std430) buffer Block2 { uint b[]; } stencilbuf;
    layout(set = 0, binding = 6) uniform texture2DArray depthTex;
    layout(set = 0, binding = 7) uniform utexture2DArray stencilTex;
    layout(local_size_x = 1, local_size_y = 1, local_size_z = 1) in;
    void main()
    {
       for (int i = 0; i < 1; ++i) {
          uint idx = ((gl_GlobalInvocationID.z * 4 + gl_GlobalInvocationID.y) * 4 + gl_GlobalInvocationID.x) * 1 + i;
          colorbuf.b[idx] = texelFetch(colorTex, ivec3(gl_GlobalInvocationID.xyz), i);
          depthbuf.b[idx] = texelFetch(depthTex, ivec3(gl_GlobalInvocationID.xyz), i).x;
       }
    }
  - `frag`

    layout(location = 0) out uvec4 col0;
    layout(set = 0, binding = 0) buffer Block { uint counter; } buf;
    layout(set = 0, binding = 3) uniform usampler2D tex;
    layout(location = 0) flat in int instanceIndex;
    layout(location = 1) flat in int readbackok;
    layout(location = 2) in float zero;
    void main()
    {
      col0.x = gl_ShadingRateEXT;
      col0.y = 0;
      col0.z = (instanceIndex << 24) | ((atomicAdd(buf.counter, 1) + 1) & 0x00FFFFFFu);
      ivec2 fragCoordXY = ivec2(gl_FragCoord.xy);
      ivec2 fragSize = ivec2(1<<((gl_ShadingRateEXT/4)&3), 1<<(gl_ShadingRateEXT&3));
      col0.w = uint(zero);
      if (((fragCoordXY - fragSize / 2) % fragSize) != ivec2(0,0))
        col0.w = 1;
      if (dFdx(gl_FragCoord.xy) != ivec2(fragSize.x, 0) || dFdy(gl_FragCoord.xy) != ivec2(0, fragSize.y))
        col0.w = (fragSize.y << 26) | (fragSize.x << 20) | (int(dFdx(gl_FragCoord.xy)) << 14) | (int(dFdx(gl_FragCoord.xy)) << 8) | 3;
      uint implicitDerivX = texture(tex, vec2(gl_FragCoord.x / textureSize(tex, 0).x, 0)).x;
      uint implicitDerivY = texture(tex, vec2(0, gl_FragCoord.y / textureSize(tex, 0).y)).x;
      if (implicitDerivX != fragSize.x || implicitDerivY != fragSize.y)
        col0.w = (fragSize.y << 26) | (fragSize.x << 20) | (implicitDerivY << 14) | (implicitDerivX << 8) | 4;
      gl_FragDepth = float(instanceIndex) / float(81);
    }
  - `vert`

    layout(push_constant) uniform PC {
            int shadingRate;
    } pc;
    layout(location = 0) in vec2 pos;
    layout(location = 0) out int instanceIndex;
    layout(location = 1) out int readbackok;
    layout(location = 2) out float zero;
    out gl_PerVertex
    {
       vec4 gl_Position;
    };
    void main()
    {
      gl_Position = vec4(pos, 0, 1);
      instanceIndex = gl_InstanceIndex;
      readbackok = 1;
      zero = 0;
    }
  - `frag_simple`
  - `vert_simple`
- `FSRTestCase::createInstance` creates a `FSRTestInstance`
- `FSRTestInstance::iterate` runs the test
  - `dsFormat` is set to `VK_FORMAT_D32_SFLOAT_S8_UINT` or
    `VK_FORMAT_D24_UNORM_S8_UINT`, depending on support
  - `atomicBuffer` is a buffer of size 4
  - `vertexBuffer` is a buffer of size 1944 (81 triangles)
  - `colorOutputBuffer` and `secColorOutputBuffer` are buffers of size 256
    - extent 4x4, with 4 uint32 channels
  - `depthOutputBufferSize` and `stencilOutputBufferSize` are buffers of size 64
    - extent 4x4, with 1 float or uint32 channel
  - `srFillBuffer` is a buffer of size 32
    - on radv, `minFragmentShadingRateAttachmentTexelSize` is 8x8
  - `cbImage` and `cbImageView` have extent 4x4 and format
    `VK_FORMAT_R32G32B32A32_UINT`
  - `dsImage` and `dsImageView` have extent 8x8 and format
    `VK_FORMAT_D32_SFLOAT_S8_UINT` or `VK_FORMAT_D24_UNORM_S8_UINT`
    - because `dsBaseMipLevel` is 1, it is mipmapped and the base extent is
      doubled
  - `dImageView` and `sImageView` are views to `dsImage` with single aspect
  - `derivImage` and `derivImageView` have extent `maxFragmentSize` and format
    `VK_FORMAT_R32_UINT`
    - it is mipmapped
    - on radv, `maxFragmentSize` is 2x2 and the image has two mipmaps
  - `sampler` is for `derivImage`
  - `descriptorSetLayouts[0]` has 8 bindings
    - `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER`
    - `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE`
    - `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER`
    - `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`
    - `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER`
    - `VK_DESCRIPTOR_TYPE_STORAGE_BUFFER`
    - `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE`
    - `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE`
  - `PipelineLayoutWrapper` is for `descriptorSetLayouts`
    - there is one push const range of size 4
  - `cs` is created from `comp`
  - `computePipeline` is created from `cs` and `pipelineLayout`
  - it then iterates over
    - `modeIdx` is `AttachmentModes`
      - `ATTACHMENT_MODE_DEFAULT`, `ATTACHMENT_MODE_LAYOUT_OPTIMAL`, etc.
    - `srTexelWidth` and `srTexelHeight` are from `minFragmentShadingRateAttachmentTexelSize` to `maxFragmentShadingRateAttachmentTexelSize`
      - on radv, they are 8x8
    - `formatIdx` is all uint formats
  - `descriptorPool` and `descriptorSets` are created for
    `descriptorSetLayouts`
  - `srImage` and `srImageView` have extent `srWidth x srHeight` and format
    `srFormat`
  - `descriptorSets` are updated
    - binding 0 is `atomicBuffer`
    - binding 1 is `cbImageView`
    - binding 2 is `colorOutputBuffer`
    - binding 3 is `derivImageView`
    - binding 4 is `depthOutputBuffer`
    - binding 5 is `stencilOutputBuffer`
    - binding 6 is `dImageView`
    - binding 7 is `sImageView`
  - `renderPass` is created
    - there are 3 attachments: `cbImageView`, `srImageView`, and `dsImageView`
    - there is 1 subpass, with all 3 attachments having layout
      `VK_IMAGE_LAYOUT_GENERAL`
      - the sr layout is `srLayout` and depends on `modeIdx`
  - `fragShader` is created from `frag`
  - `vertShader` is created from `vert`
  - `cmdPool` and `cmdBuffer` are the cmdbuf
  - `secVertexBuf` is created but unused
  - `fragSimpleShader` is created from `frag_simple` but unused
  - `vertSimpleShader` is created from `vert_simple` but unused
  - `pipelineLayout1` is created but unused
  - `vkBeginCommandBuffer`
  - render
    - `preRenderCommands`
      - it transitions all images to `VK_IMAGE_LAYOUT_GENERAL`
      - it clears `derivImage` to `(1<<lv, 0, 0, 0)`
      - it clears `cbImage` to `(0, 0, 0, 0)`
      - it clears `dsImage` to `(0.0f, 0)`
      - it initializes `srFillBuffer` to `SanitizeRate`
      - it `vkCmdCopyBufferToImage` from `srFillBuffer` to `srImage`
      - barriers
    - `beginLegacyRender` begins the render pass
    - `drawCommands`
      - it binds `pipelineLayout` and `descriptorSets` to
        `VK_PIPELINE_BIND_POINT_GRAPHICS`
      - it creates and binds `pipelines[0]` to
        `VK_PIPELINE_BIND_POINT_GRAPHICS`
      - for each of `NUM_TRIANGLES` (81) triangles
        - it `vkCmdBindVertexBuffers` `vertexBuffer` with the right offset
        - it `vkCmdPushConstants` `PrimIDToPrimitiveShadingRate`
          - `PrimIDToPipelineShadingRate`
            - `width = 1 << ((primID/9) % 3)`
            - `height = 1 << (((primID/9)/3) % 3)`
            - `ShadingRateExtentToEnum` encodes the w/h using 4 bits
          - `ShadingRateEnumToExtent` decodes the w/h
        - it `vkCmdSetFragmentShadingRateKHR`
          - the two combiner ops are keep and replace; that is, the final
            shading rate is determined by the contents of `srImage`
        - it `vkCmdDraw` with `gl_InstanceIndex` being the prim id
    - `vkCmdEndRenderPass`
  - `vkCmdPipelineBarrier`
  - it binds `computePipeline` and calls `vkCmdDispatch`
  - `vkCmdPipelineBarrier`
  - `vkEndCommandBuffer`
  - it checks the results
    - it iterates over `numColorLayers`, `framebufferDim`, and `samples`
    - `colorptr` is mapped `colorOutputBuffer`
    - `depthptr` is mapped `depthOutputBuffer`
    - `fillPtr` is mapped `srFillBuffer`
- for this specific test on radv,
  - `framebufferDim` is `4x4`
  - `srTexelWidth x srTexelHeight` is `8x8`
  - `srWidth x srHeight` is `1x1`
  - pipeline rate is set by `vkCmdSetFragmentShadingRateKHR` and varies for
    each 9 triangles
  - primitive rate is set by `gl_PrimitiveShadingRateEXT` and is not set
    - only gs/ms sets it to push const
  - attachment rate is set by `srImage` contents and is 0
    - `preRenderCommands` fills `0` to `srImage`
  - the fs outputs
    - `gl_FragDepth = float(instanceIndex) / float(81);`, where
      `instanceIndex` is the triangle primid from 0 to 80
    - `col0.x = gl_ShadingRateEXT;`
    - `col0.y = 0;`
    - `col0.z = (instanceIndex << 24) | ((atomicAdd(buf.counter, 1) + 1) & 0x00FFFFFFu);`
    - `col0.w` is the error code; non-zero means failure

## Test Case: `dEQP-VK.shader_object.performance.binary_memcpy`

- `ShaderObjectBinaryPerformanceCase` is the test case
  - `BinaryType` is `BINARY_MEMCPY`
- `ShaderObjectBinaryPerformanceInstance` is the test instance
- `ShaderObjectBinaryPerformanceInstance::iterate`
  - `vkCreateShadersEXT` creates a `VkShaderEXT` from spirv
  - `vkGetShaderBinaryDataEXT` queries the binary
  - `vkCreateShadersEXT` creates a `VkShaderEXT` from the binary
  - `vkCreateBuffer` creates a `VkBuffer`
  - mempcy to copy the binary to the buffer
  - it then validates that `vkCreateShadersEXT` from the binary is at most 50%
    slower than mempcy

## Test Case: `dEQP-VK.shader_object.performance.draw_static_pipeline`

- `ShaderObjectPerformanceCase` is the test case
  - `DrawType` is `DRAW`
  - `TestType` is `DRAW_STATIC_PIPELINE`
- `ShaderObjectPerformanceInstance` is the test instance
- `ShaderObjectPerformanceCase::initPrograms` creates very simple vs and fs
- `ShaderObjectPerformanceInstance::iterate`
  - `ShaderObjectPerformanceInstance::draw` calls `vkCmdDraw*` and measures
    the cpu time
  - it binds the ESOs and calls `draw` to measure the cpu time
  - it tries again by binding the pipeline and calls `draw` to measure
    the cpu time
  - it then validates that ESOs are at most 50% slower than pipeline regarding
    cpu overhead

## Test Case: `dEQP-VK.synchronization.timeline_semaphore.wait.poll_signal_from_device`

- `PollTestInstance`
- it creates 1 `VkFence` and 100 timeline `VkSemaphore`
- `deviceSignal` is called 100 times to signal the semaphores to different
  values
  - the fence is used to wait for the last submit

## Test Case: `dEQP-GLES31.functional.copy_image.mixed.viewclass_128_bits_mixed.rgba32ui_srgb8_alpha8_astc_4x4_khr.texture2d_to_texture2d`

- `CopyImageTests::init` calls `addCopyTests`
  - `srcFormat` is `GL_RGBA32UI`
  - `dstFormat` is `GL_COMPRESSED_SRGB8_ALPHA8_ASTC_4x4`
  - `srcSize` is `(129, 127)`, 8 levels
  - `dstSize` is `(132, 124)`, 8 levels
- `CopyImageTest::iterate`
  - iteration 1
    - `CopyImageTest::createImagesIter` creates and inits src and dst images
      - `genImage` calls `gl.texImage2D` or `gl.compressedTexImage2D` with
        generated texel data
        - because this is astc, `genTexel` randomly picks one of 8 possible
          values
    - `CopyImageTest::renderSourceIter` renders the src image to the current fb
      - the origin is picked by `RandomizedRenderGrid`
  - iteration 2
    - `CopyImageTest::renderDestinationIter` renders the dst image to the
      current fb
  - iteration 3
    - `CopyImageTest::copyImageIter` copies random (aligned) regions from the
      src to the dst with `gl.copyImageSubData`
      - `copyImageData` simulates the copy on cpu for verification
    - `CopyImageTest::verifySourceIter` renders the src image to the current
      fb and `glu::readPixels` to verify
  - iteration 4
    - `CopyImageTest::verifyDestinationIter` renders the dst image to the
      current fb and `glu::readPixels` to verify
    - `CopyImageTest::destroyImagesIter` destroys both images
  - iteration 5
    - `CopyImageTest::createImagesIter`
    - `CopyImageTest::copyImageIter`
    - `CopyImageTest::verifySourceIter`
  - iteration 6
    - `CopyImageTest::verifyDestinationIter`
    - `CopyImageTest::destroyImagesIter`

- under angle, and when simplified to 1 level and 1 copy,
  - `createImagesIter`
    - because the src image size is `(129, 127)`, there is a
      `vkCmdCopyBufferToImage` with the same extent
    - because the dst image size is `(132, 124)`, there is a
      `vkCmdCopyBufferToImage` with the same extent
  - `copyImageIter`
    - because the copy is from `(114, 114)` to `(0, 0)` of src size
      `(15, 13)`, there is a `vkCmdCopyImage` with
      - `srcOffset` is `(114, 114)`
      - `dstOffset` is `(0, 0)`
      - `extent` is `(15, 13)`
  - `verifyDestinationIter`
    - there is a render pass with render area `(400, 300)` and a
      `vkCmdDrawIndexed`
    - there is a `vkCmdCopyImageToBuffer` with `imageOffset` of `(1, 113)` and
      `imageExtent` of `(264, 124)`
  - `destroyImagesIter`

## Test Case: `dEQP-GLES3.functional.fbo.msaa.4_samples.depth_component16`

- `FboMultisampleTests::init` creates a `BasicFboMultisampleCase` with
  - `m_colorFormat = GL_RGBA8`
  - `m_depthStencilFormat = GL_DEPTH_COMPONENT16`
  - `m_size = IVec2(119, 131)`
  - `m_numSamples = 4`
  - `FboTestCase`
    - `m_viewportWidth = 128`
    - `m_viewportHeight = 128`
- `FboTestCase::iterate`
  - it picks a random `(x, y)` in the render target window
  - `BasicFboMultisampleCase::preCheck` checks if the test is supported
  - it creates a `sglr::GLContext` with viewport `(x, y, 128, 128)`
  - `BasicFboMultisampleCase::render`
    - it creates 2 FBOs of size 119x131
      - both have color and depth
      - the first is mutisampled and the second is single-sampled
    - draw
      - depth is cleared to 1.0
      - draw a full fbo quad with z goes from -1.0 to 1.0
        - the default depth function is `GL_LESS`
      - draw random 8 quads with depth function set to `GL_ALWAYS`
      - blit to resolve samples
    - visualize
      - it draws 8 full fbo quads with different but fixed z
      - the depth func is `GL_ELSS` and the depth write is disabled
    - it reads back the color values for comparision later
  - it creates a `sglr::ReferenceContext`
  - `BasicFboMultisampleCase::render` again
  - `BasicFboMultisampleCase::compare` calls `FboTestCase::compare` to compare
    the result and the reference surfaces
