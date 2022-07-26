Android CTS
===========

## CTS

- `./cts-tradefed run commandAndExit cts-dev -m <module> -t <CLASS>#<METHOD>`
  - this runs the primary abi
    - for other abis, specify `--abi` or `-a`
      - `x86_64`
      - `arm64-v8a`
  - `-t` can be replaced by
    `--module-arg 'CtsGraphicsTestCases:include-filter:android.graphics.cts.VulkanFeaturesTest*'`
- <https://source.android.com/compatibility/cts/downloads.html>
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
- `CtsCameraTestCases`
  - `--module-arg 'CtsCameraTestCases:include-filter:android.hardware.cts.CameraGLTest#*'`
    - 2m
