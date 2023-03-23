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

## Test Case: `dEQP-VK.renderpass2.depth_stencil_resolve.image_2d_49_13.samples_2.d16_unorm_s8_uint.depth_none_stencil_zero_testing_stencil`

- renderpass
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
- pipeline
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
