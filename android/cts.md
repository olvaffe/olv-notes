Android CTS
===========

## Build

- <https://source.android.com/docs/compatibility/cts/development>
  - `repo init ...`
  - `. build/envsetup.sh`
  - `lunch aosp_x86_64`
  - `m cts`
  - `cd out/host/linux-x86/cts/android-cts`
- branches
  - `android10-tests-dev`, `android11-tests-dev`, and so on are for
    developments
    - `android11-tests-dev` requires `libncurses.so.5` and `libtinfo.so.5`
      from the host.  Create symlinks if your distro only has `libncurses6`
  - `android10-tests-release`, `android11-tests-release`, and so on merge
    from the dev branches to make quarterly releases
- old branches
  - `pie-cts-dev` and `pie-cts-release`
  - iirc, `prebuilts/misc/linux-x86/flex` might be too old
    - copy the one in the host to here

## Run

- <https://source.android.com/compatibility/cts/downloads.html>
- permissions
  - `adb shell settings put global verifier_verify_adb_installs 0`
  - `adb shell settings put global package_verifier_enable 0`
- `./cts-tradefed run commandAndExit cts-dev -m <module> -t <CLASS>#<METHOD>`
  - this runs the primary abi
    - for other abis, specify `--abi` or `-a`
      - `x86_64`
      - `arm64-v8a`
  - `-t` can be replaced by
    `--module-arg 'CtsGraphicsTestCases:include-filter:android.graphics.cts.VulkanFeaturesTest*'`

## Sources

- <https://android.googlesource.com/platform/tools/tradefederation/>
  - <https://android.googlesource.com/platform/tools/tradefederation/+/refs/heads/master/src/com/android/tradefed/testtype/suite/BaseTestSuite.java>
  - <https://android.googlesource.com/platform/tools/tradefederation/+/refs/heads/master/src/com/android/tradefed/command/CommandOptions.java>
- <https://android.googlesource.com/platform/cts/>
  - <https://android.googlesource.com/platform/cts/+/refs/heads/master/tools/cts-tradefed/res/config/cts-dev.xml>
  - for fastest runs, use `cts-dev` plan

## graphics-related modules

- `CtsGraphicsTestCases`
  - 15m
  - `-t android.graphics.cts.*`
  - `-t android.graphics.drawable.*`
  - `-t android.graphics.fonts.*`
  - `-t android.graphics.text.*`
- `CtsNativeHardwareTestCases`
  - 1m
  - `-t android.hardware.nativehardware.cts.AHardwareBufferNativeTests*`
  - `-t android.hardware.nativehardware.cts.HardwareBufferVrTest*`
- `CtsDeqpTestCases`
  - `-t dEQP-EGL.*`
    - 15m
  - `-t dEQP-GLES2.*`
    - 10m
  - `-t dEQP-GLES3.*`
    - 45m
  - `-t dEQP-GLES31.*`
    - 40m
  - `-t dEQP-VK.*`
    - 8h?
  - logcat says `Writing test log into /sdcard/TestLog.qpa`
- `CtsSkQPTestCases`
  - 10m
  - `-t org.skia.skqp.SkQPRunner*`
- `CtsUiRenderingTestCases`
  - 2m
  - `-t android.uirendering.cts.testclasses.*`
- `CtsViewTestCases`
  - `--module-arg 'CtsViewTestCases:include-filter:android.view.cts.PixelCopyTest#*'`
    - 2m
  - `--module-arg 'CtsViewTestCases:include-filter:android.view.cts.SurfaceViewSyncTest#*'`
    - 10m
  - `--module-arg 'CtsViewTestCases:include-filter:android.view.cts.TextureViewTest#*'`
    - 1s
  - `--module-arg 'CtsViewTestCases:include-filter:android.view.cts.ASurfaceControlTest#*'`
    - hours?
- `CtsMediaTestCases`
  - `--module-arg 'CtsMediaTestCases:include-filter:android.media.cts.DecodeAccuracyTest*'`
    - 5m
  - `--module-arg 'CtsMediaTestCases:include-filter:android.media.cts.EncodeDecodeTest*'`
- `CtsCameraTestCases`
  - `--module-arg 'CtsCameraTestCases:include-filter:android.hardware.cts.CameraGLTest#*'`
    - 2m
- `CtsNNAPITestCases`
- `CtsGpuToolsHostTestCases`
- `CtsOpenGLTestCases`
- `CtsVideoTestCases`
-  this seems like a good starting point
  - `CtsCameraTestCases`
  - `CtsDeqpTestCases`
  - `CtsGpuMetricsHostTestCases`
  - `CtsGpuProfilingDataTestCases`
  - `CtsGpuToolsHostTestCases`
  - `CtsGraphicsTestCases`
  - `CtsHardwareTestCases`
  - `CtsMediaCodecTestCases`
  - `CtsMediaDecoderTestCases`
  - `CtsMediaEncoderTestCases`
  - `CtsMediaV2TestCases`
  - `CtsNativeHardwareTestCases`
  - `CtsOpenGlPerf2TestCases`
  - `CtsOpenGlPerfTestCases`
  - `CtsOpenGLTestCases`
  - `CtsSkQPTestCases`
  - `CtsUiAutomationTestCases`
  - `CtsUiRenderingTestCases`
  - `CtsUiRenderingTestCases27`
  - `CtsVideoTestCases`

## Vulkan Requirements

- terms
  - CDD: Compatibility Definition Document
    - defines the android api for app devs
    - ensures app compat across android devices
  - VSR: Vendor Software Requirements
    - defines the kernel and hal interfaces
    - ensures AOSP system images work across android devices
  - GMS: Google Mobile Services
    - a collection of proprietary apps and services from google
    - oem must sign MADA (Mobile Application Distribution Agreement) to
      license
  - GMS Requirements
    - defines requirements a device must comply to preload GMS
    - enforced by MADA
  - CTS: Compatibility Test Suite
    - tests CDD compliance
  - VTS: Vendor Test Suite
    - tests VSR compliance
  - GTS: GMS Test Suite
    - tests GMS compliance
- Vulkan features
  - <https://developer.android.com/reference/android/content/pm/PackageManager>
  - `FEATURE_VULKAN_HARDWARE_VERSION`
    - 1.1 implies
      - `VK_ANDROID_external_memory_android_hardware_buffer`
      - `SYNC_FD`
      - `samplerYcbcrConversion`
    - may be software-based
  - `FEATURE_VULKAN_HARDWARE_LEVEL`
    - 0 implies `textureCompressionETC2`
    - 1 implies
      - `textureCompressionASTC_LDR`
      - and many more
  - `FEATURE_VULKAN_HARDWARE_COMPUTE`
    - 0 implies `VK_KHR_variable_pointers` and more
- Android Q+ requires Vulkan 1.1
  - <https://android-developers.googleblog.com/2019/05/whats-new-in-android-q-beta-3-more.html>
  - that's probably required by GTS/VTS for new 64-bit devices
- Android 13 CDD
  - <https://source.android.com/docs/compatibility/13/android-13-cdd>
  - strongly recommend Vulkan 1.3
  - strongly recommend Android Baseline 2021 profile

## `android.graphics.cts.MediaVulkanGpuTest#testMediaImportAndRendering`

- `loadMediaAndVerifyFrameImport`
  - calls `VkInit::init` to initialize vulkan
  - calls `VkImageRenderer::init` to create a bunch of vk objects
    - image size is `1920x1080`
    - format is `VK_FORMAT_R8G8B8A8_UNORM`
    - this creates various vk objects
      - image/view/renderpass for rendering
      - pipeline with `passthrough_vsh.spv` and `passthrough_fsh.spv`
        - vs is `gl_Position = pos; texcoord = attr;`
        - fs is `uFragColor = texture(tex, texcoord)`
      - buffer for readback
  - calls `ImageReaderHelper::initImageReader` to initialize a image reader
    - image size is `1920x1080`
    - fomrat is `AIMAGE_FORMAT_YUV_420_888`
    - usage is `AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE`
    - image count is 3
  - calls `MediaHelper::init` to initialize the media
    - this will decode `test_video.mp4` to the image reader
  - calls `VkAHardwareBufferImage::init` to export an AHB from the image and
    to a vk image/view/sampler for the AHB
  - calls `VkImageRenderer::renderImageAndReadback` to convert yuv to rgb
    - `vkCmdDraw` with a passthrough pipeline to sample the yuv image and to
      draw to the rgb image
    - `vkCmdCopyImageToBuffer` to copy from the rgb image to the readback
      buffer
  - compares the readback buffer to the reference buffer
- minigbm on grunt sees these allocations
  - `amdgpu_create_bo_linear(1048576, 1, 0x20203852, 0x2a00)`
    - `R8  `
    - `SW_READ_OFTEN | SW_WRITE_OFTEN | HW_VIDEO_DECODER`
  - `amdgpu_create_bo_linear(1048576, 1, 0x20203852, 0x2a00)`
  - `amdgpu_create_bo_linear(1048576, 1, 0x20203852, 0x2a00)`
  - `amdgpu_create_bo_linear(1048576, 1, 0x20203852, 0x2a00)`
  - `amdgpu_create_bo_linear(1048576, 1, 0x20203852, 0x2a00)`
  - `amdgpu_create_bo_linear(1048576, 1, 0x20203852, 0x2a00)`
  - `amdgpu_create_bo_linear(1048576, 1, 0x20203852, 0x2a00)`
  - `amdgpu_create_bo_linear(1920, 1088, 0x3231564e, 0x2020)`
    - `NV12`
    - `TEXTURE | HW_VIDEO_DECODER`
    - stride is `2048` because minigbm thinks gpu requires 256-byte alignment
    - plane 0 size is `2048 * 1088 = 2228224`
    - plane 1 offset is `2228224` and size is `1114112`
- radv on grunt sees these images/memories
  - `radv_GetAndroidHardwareBufferPropertiesANDROID`
    - `VkAndroidHardwareBufferPropertiesANDROID`
      - `allocationSize` is 3342336
      - `memoryTypeBits` is 0x2d
    - `VkAndroidHardwareBufferFormatProperties2ANDROID`
      - `format` and `externalFormat` are `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM`
        - radv sees ahb format `AHARDWAREBUFFER_FORMAT_Y8Cb8Cr8_420`, which is
          translated from `AIMAGE_FORMAT_YUV_420_888` by the media stack
        - external format is driver-defined and mesa simply uses `VkFormat`
      - `formatFeatures` is `0xc00000`
        - `VK_FORMAT_FEATURE_SAMPLED_IMAGE_BIT`
        - `VK_FORMAT_FEATURE_TRANSFER_SRC_BIT`
        - `VK_FORMAT_FEATURE_TRANSFER_DST_BIT`
        - `VK_FORMAT_FEATURE_MIDPOINT_CHROMA_SAMPLES_BIT`
        - `VK_FORMAT_FEATURE_SAMPLED_IMAGE_YCBCR_CONVERSION_LINEAR_FILTER_BIT`
        - `VK_FORMAT_FEATURE_DISJOINT_BIT`
        - `VK_FORMAT_FEATURE_COSITED_CHROMA_SAMPLES_BIT`
      - ycbcr-related fields are hardcoded
  - `radv_CreateImage`
    - `format` is `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM` which is from
      `VkExternalFormatANDROID`
    - size is 1920x1088
    - `image->plane_count` is 2
  - `radv_AllocateMemory`
    - `VkImportAndroidHardwareBufferInfoANDROID` specifies the ahb to import
    - `VkMemoryDedicatedAllocateInfo` is used as required by ahb
    - plane 0
      - offset 0
      - format `VK_FORMAT_R8_UNORM`
    - plane 1
      - offset 2228224
      - format `VK_FORMAT_R8G8_UNORM`

## `android.media.encoder.cts.VideoEncoderTest#testEncode`

- `VideoEncoderTest::input`
  - the test is parameterized and `input` returns a list of parameters
  - for each encoder returned by `MediaUtils.getEncoderNamesForMime`
    - `TEST_MODE_DETAILED` tests common sizes
    - `TEST_MODE_INTRAREFRESH` tests `480x360`
    - `TEST_MODE_SPECIFIC` tests supported sizes
      - `EncoderSize` queries codec supported sizes
    - `flexYuv` can be true or false
- `VideoEncoderTest::VideoEncoderTest`
  - `getEncHandle` returns an `Encoder`
  - e.g., `[31(c2.v4l2.avc.encoder_video/avc_66x2,158_false_TEST_MODE_SPECIFIC)]`
    - `encoderName` is `c2.v4l2.avc.encoder`
    - `mime` is `video/avc`
    - `width` is 66
    - `height` is 2158
    - `flexYuv` is false
    - `mode` is `TEST_MODE_SPECIFIC`
- `VideoEncoderTest::testEncode` picks the test depending on the mode
- `VideoEncoderTest::specific` calls `Encoder::testSpecific`
  - `processor` is `VideoProcessor` or `SurfaceVideoProcessor` depending on
    `flexYuv`
- `SurfaceVideoProcessor::processLoop`
  - `open` initializes `mExtractor`, `mDecFormat`, and `mEncodedStream` from
    `video_480x360_mp4_h264_871kbps_30fps.mp4`
  - `initCodecsAndConfigureEncoder` initializes `mDecoder` and `mEncoder`
  - `mEncSurface` and `mDecSurface` are created
  - when the decoder can receive the next buffer, `onInputBufferAvailable`
    calls `fillDecoderInputBuffer` to read a buffer from the extractor and
    queue it into the decoder
  - after the decoder decodes a buffer, `onOutputBufferAvailable` calls
    `renderDecodedBuffer` to set `mFrameAvailable`
  - the process loop calls `mDecSurface.latchImage` and
    `mDecSurface.drawImage` to draw the decoded image to the encoder's input,
    and calls `mEncSurface.swapBuffers` to start encoding
  - after the encoder encodes a buffer, `onOutputBufferAvailable` calls
    `emptyEncoderOutputBuffer` to add the buffer to `mEncodedStream`
- `VideoProcessorBase::playBack` decodes `mEncodedStream` and plays the
  buffers
