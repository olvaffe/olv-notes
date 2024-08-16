CrOS Autotest
=============

## Overview

- <https://chromium.googlesource.com/chromiumos/third_party/autotest>
- history
  - it was forked from <https://github.com/autotest/autotest>
  - the upstream devs have moved to
    <https://github.com/avocado-framework/avocado>
- `test_that $DUT $TEST_NAME` is the main cmdline
- client-side tests lives under `client/site_tests`
  - these are tests that run entirely on the dut
  - a host in this context means the dut
- server-side tests lives under `server/site_tests`
  - these are tests that run on the host (i.e., the system that runs
    `autoserv`) which controls a dut
  - a host in this context means the host
- test suites appear to live under `test_suites`
  - each test suite is just a servier-side test
- a test is defined by its control file
  - `NAME` is the test name
  - `TEST_TYPE` is either `Client` or `Server`
  - `ATTRIBUTES` is a comma-separated list of attrs
    - `suite:suite-a, suite:suite-b` means the test should be run as a part
      of both `suite-a` and `suite-b`

## Test Suites

- `src/third_party/autotest/files/test_suites/control.graphics_per-week`
  - this defines test suite `graphics_per-week`
- `src/third_party/autotest/files/client/site_tests/graphics_parallel_dEQP/control.vk.0`
  - this test has `suite:graphics_per-week` and is a member of the test suite
  - it runs `graphics_parallel_dEQP` autotest with the specified args
- `src/third_party/autotest/files/server/site_tests/tast/control.graphics-weekly`
  - this test has `suite:graphics_per-week` and is a member of the test suite
  - it runs `tast` autotest with the specified args
    - this in turn runs tast tests that match the specified args

## `test_that DUT suite:bvt-tast-cq`

- the test suite is defined in `test_suites/control.bvt-tast-cq`
- these server-side tests have `suite:bvt-tast-cq` in their `ATTRIBUTES`
  - `server/site_tests/tast/control.critical-android-shard-*`
    - `group:mainline`, `name:crostini.*`
    - `!informational`, `!group:crostini_slow`
  - `server/site_tests/tast/control.critical-crostini-shard-*`
    - `group:mainline`, `dep:android*`
    - `!informational`, `!name:crostini.*`
  - `server/site_tests/tast/control.critical-chrome-shard-*`
    - `group:mainline`, `dep:chrome`
    - `!informational`, `!name:crostini.*`, `!dep:android*`
  - `server/site_tests/tast/control.critical-system-shard-*`
    - `group:mainline`,
    - `!informational`, `!name:crostini.*`, `!dep:android*`, `!dep:chrome`
  - roughly speaking, there is a total of 15 server-side tests and they runs
    most `group:mainline` that are not `!informational`
- `test_that` artifacts
  - `debug/test_that.*` are `test_that` logs at different levels
  - `test_report.log` provides a high-level report of the server-side tests
  - `result-*` are logs for all 15 server-side tests
- server-side test artifacts
  - `debug/autoserv.*` are `autoserv` logs at different levels
  - `status.log` provides a high-level report of the client-side tests
  - `sysinfo` provides DUT system logs
  - `tast` provides tast logs
- tast test artifacts
  - `debug/tast.*` are `tast` logs at different levels
  - `sysinfo` provides DUT system logs
  - `tast` provides tast logs
  - `results` provides test results
    - `tests` are logs for each tast tests

## Graphics Tests

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

## `cheets_CTS_R`

- `src/third_party/autotest/files/server/site_tests/cheets_CTS_R`
- `cheets_CTS_R.py`
  - `LATEST` is official cts
  - `DEV` is preview cts
- `control.internal.arm.CtsDeqp*`
  - `cheets_CTS_R.internal.arm.CtsDeqp.32` and `cheets_CTS_R.internal.arm.CtsDeqp.64` use DEV
  - the rest uses LATEST
- critical CTS
  - `cheets_CTS_R.internal.arm.CtsDeqpTestCases.dEQP-EGL`, 35m
  - `cheets_CTS_R.internal.arm.CtsDeqpTestCases.dEQP-GLES2`, 35m
  - `cheets_CTS_R.internal.arm.CtsDeqpTestCases.dEQP-GLES3`, 200m
  - `cheets_CTS_R.internal.arm.CtsDeqpTestCases.dEQP-GLES31`, 220m
- fast CTS
  - `cheets_CTS_R.internal.arm.CtsCamera`, 10m
  - `cheets_CTS_R.internal.arm.CtsGpu`, 15m
  - `cheets_CTS_R.internal.arm.CtsGraphics`, 30m
  - `cheets_CTS_R.internal.arm.CtsNNAPI`, 15m
  - `cheets_CTS_R.internal.arm.CtsNative`, 20m
  - `cheets_CTS_R.internal.arm.CtsOpenG`, 15m
  - `cheets_CTS_R.internal.arm.CtsSkQP`, 20m
  - `cheets_CTS_R.internal.arm.CtsUi`, 10m
- slow CTS
  - `cheets_CTS_R.internal.arm.CtsView`, 160m
  - `cheets_CTS_R.internal.arm.CtsVideo`, 260m
  - `cheets_CTS_R.internal.arm.CtsMediaStressTestCases`, 300m
  - `cheets_CTS_R.internal.arm.CtsMediaTestCases.64`, 900m
  - `cheets_CTS_R.internal.arm.CtsMediaV2TestCases`
- what are `suite:arc-cts`, `suite:arc-cts-r`, `suite:arc-cts-unibuild`,
  `suite:arc-cts-qual`, and `suite:arc-cts-hardware`?
  - `infra/suite_scheduler/generated_configs/suite_scheduler.ini`
  - `arc-cts-qual` and `arc-cts-hardware`
    - run on Mon, Wed, and Fri at hour 0 for beta channel
    - run on Sat at hour 0 for stable channel
    - run on Sun at hour 6 for dev channel
    - timeout is 30 hours
  - `arc-cts-r`
    - run on selected days at hour 0 for dev channel
      - Tue: {eve,kukui,trogdor,zork}-arc-r
      - Thu: {kukui,trogdor}-arc-r
      - Sat: kukui-arc-r
      - Sun: zork-arc-r
    - timeout is 43 hours
  - `arc-cts`
    - run on Mon, Wed, and Fri at hour 11 for dev channel
    - run on Tue, Thu, and Sat at hour 0 for dev channel
    - timeout is 30 hours

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
