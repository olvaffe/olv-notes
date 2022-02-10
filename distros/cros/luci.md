Chrome OS CI
============

## CI

- <https://ci.chromium.org/>
- projects
  - chromium <https://ci.chromium.org/p/chromium>
  - fuchsia <https://ci.chromium.org/p/fuchsia>
  - angle <https://ci.chromium.org/p/angle>
- builders
  - each project can have many builders
  - angle's android-arm64-rel
    <https://ci.chromium.org/p/angle/builders/ci/android-arm64-rel>
  - fuchsia's clang-linux-arm64
    <https://ci.chromium.org/p/fuchsia/builders/toolchain.ci/clang-linux-arm64>
  - chromium's Linux Builder
    <https://ci.chromium.org/p/chromium/builders/ci/Linux%20Builder>
- builds
  - each builder can have many builds
  - build 140196 of chromium's Linux Builder
    <https://ci.chromium.org/ui/p/chromium/builders/ci/Linux%20Builder/140196/overview>
- groups
  - a group groups related builders together
  - a group can present a console view
  - a builder can belong to multiple groups

## autotest and tast

- autotest
  - <https://chromium.googlesource.com/chromiumos/third_party/autotest/>
  - `test_that $DUT $TEST_NAME`
- autotest tests
  - `AUTHOR = "chromeos-gfx"`
    - `graphics_Check` takes some screenshots and verifies they are not black.
      Stops ui and draws/readbacks to verify.
    - `graphics_Chrome` runs `ozone_gl_unittests` that tests dma-buf and
      GLImageEGL
    - `graphics_GLAPICheck` runs `wflinfo` and verifies GL/GLES versions and
      extensions
    - `graphics_GLBench` runs `glbench`
    - `graphics_GLMark2` runs `glmark2`
    - `graphics_Gbm` runs minigbm unit tests
    - `graphics_HwOverlays` loads a test webpage and verifies at least two
      overlays are active
    - `graphics_Idle` naviates to a static webpage and verifies that the GPU
      is clocked down
    - `graphics_KernelMemory` verifies drm info is available in sysfs
    - `graphics_LibDRM` runs `modetest` and `kmstest`
    - `graphics_PerfControl` verifies the machine can be cooled down
    - `graphics_Power` runs `power_Test`
    - `graphics_SanAngeles` runs a benchmark
    - `graphics_Stress` navigates to some webpages, restarts chrome, etc
    - `graphics_VTSwitch`
    - `graphics_VideoRenderingPower`
    - `graphics_WebGLAquarium`
    - `graphics_WebGLManyPlanetsDeep`
    - `graphics_dEQP` runs dEQP
    - `graphics_parallel_dEQP` runs dEQP in parallel
    - `graphics_per-build`
    - `graphics_per-day`
    - `graphics_per-week`
    - `vmtest_informational[1-4]`
  - `cheets_CTS_R` `CtsGraphicsTestCases` and `CtsDeqpTestCases`
- tast
  - <https://chromium.googlesource.com/chromiumos/platform/tast/>
  - <https://chromium.googlesource.com/chromiumos/platform/tast-tests>
- tast tests
  - `tast list <dut>`
    - `graphics.DEQP` runs a very small dEQP subset
    - `graphics.DRM` runs various `drm_tests`
    - `graphics.*`
    - `crostini.*`
    - `arc.Boot.vm`
    - `arc.DEQP.vm`
  - `tast list -buildbundle crosint <dut>`
