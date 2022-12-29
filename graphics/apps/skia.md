Skia
====

## Build

- <https://skia.org/docs/user/build/>
  - `git clone https://skia.googlesource.com/skia.git`
  - `cd skia`
  - `./tools/git-sync-deps`
  - `./tools/install_dependencies.sh`
  - `./bin/gn gen out --args='is_official_build=false'`
  - `ninja -C out`
- args
  - list
    - `gn args --list out`
    - `gn/BUILDCONFIG.gn`
    - `gn/skia.gni`
  - gl/gles
    - `skia_use_gl = true` to enable GL or GLES backend
    - `skia_use_egl = true` to pick EGL/GLES on Linux
      - does not affect android
    - `skia_use_x11 = false` to pick GLX/GL on Linux
  - vulkan
    - `skia_use_vulkan = true` to enable VK backend
  - disable font
    - `skia_enable_fontmgr_android = false`
    - `skia_enable_fontmgr_empty = true`
  - disable optional features
    - `skia_enable_pdf = false`
  - cross-compile
    - `target_os = "linux"`
    - `target_cpu = "arm64"`
    - `target_cc = "/usr/bin/aarch64-linux-gnu-gcc"`
    - `target_cxx = "/usr/bin/aarch64-linux-gnu-g++"`
    - `extra_cflags = [ "--sysroot=/sysroot-arm64", "-Wno-error" ]`
    - `extra_ldflags = [ "--sysroot=/sysroot-arm64" ]`
  - misc
    - `cc_wrapper = "ccache"`
- skia on angle
  - <https://skia.org/docs/user/special/angle/>
- skia on vulkan
  - <https://skia.org/docs/user/special/vulkan/>

## Tests

- terms
  - gm stands for golden master test suite
  - dm stands for dungeon master and runs unit tests and gms
    - first commit says "dm is like gm, but faster and with fewer features"
  - fm was an attemp to become better dm?
    - first commit says "FM, a dumber new testing tool"
- <https://skia.org/docs/dev/testing/>
  - `./out/fm --resourcePath resources --sources arithmode --backend vk`
    - `--listTests`
    - `--listGMs`
  - `./out/dm --resourcePath resources --src gm --config vk --verbose`
  - see below for skqp
- tests
  - `DEF_GANESH_TEST_*`
    - defines a test of type `skiatest::TestType::kGanesh`
    - a gpu unit test
    - on older branches, this is `DEF_GPUTEST`
  - `DEF_TEST`
    - defines a test of type `skiatest::TestType::kCPU`
    - a cpu unit test
  - `DEF_GM`
    - defines a GM test

## SkQP

- tot is from <https://skia.googlesource.com/skia.git>
- cts branches are from <https://android.googlesource.com/platform/external/skqp>
- tests
  - on tot, skqp only runs gpu (ganesh) unit tests
    - see `get_unit_tests`
  - on `android11-tests-release`, it runs
    - gm tests (`gles_`, `vk_`, and `gl_`)
      - see `get_render_tests`
    - gpu unit tests (`unitTest_`)
      - see `get_unit_tests`
  - on `pie-cts-release`, it runs
    - gm tests (`SkiaGM_gles.*`, `SkiaGM_vk.*`, and `SkiaGM_gl.*`)
    - all unit tests (`Skia_Unit_Tests.*`)
    - see `register_skia_tests`
- linux build on tot
  - edit `tools/skqp/src/skqp.h` to set `fEnforcedAndroidAPILevel` to 99
    - otherwise, all tests are skipped
  - `./out/skqp . report` to run all tests
    - `.` is the asset dir: skqp looks for `resources` under the asset dir
    - `report` is for test results
- linux build on `android11-tests-release`
  - install `python-is-python2`
  - edit `DEPS` and `BUILD.gn` to remove all mentions of `Nima`
  - `tools/git-sync-deps`
  - use args from `tools/skqp/skqp_gn_args.py`, `tools/skqp/generate_gn_args`,
    and `tools/skqp/gn_to_bp.py`, and add
    - `skia_use_x11 = false`
    - `skia_use_egl = true`
    - `extra_cflags = [ "-Wno-error" ]`
  - fix trivial compile errors
    - remove duplicated definitions from `include/config/SkUserConfig.h`
    - add `#include <limits>` to
      `third_party/externals/spirv-tools/source/validate_cfg.cpp`
    - fix aarch64-specific issues by using the slow paths (or use clang)
      - `src/opts/SkRasterPipeline_opts.h`
  - `ninja skqp`
    - note that many tests do not build in this config because we disable pdf
      and skottie
  - `./out/skqp platform_tools/android/apps/skqp/src/main/assets report '^vk_'`
- linux build on `pie-cts-release`
  - similiar to on `android11-tests-release`
  - install `python-is-python2`
  - `tools/git-sync-deps`
  - use args from `tools/skqp/generate_gn_args` and `tools/skqp/gn_to_bp.py`,
    and add
    - `skia_use_egl = true`
    - `extra_cflags = [ "-Wno-error" ]`
  - fix trivial compile errors
    - remove duplicated definitions from `include/config/SkUserConfig.h`
    - fix aarch64-specific issues by using the slow paths (or use clang)
      - `src/jumper/SkJumper_stages.cpp`
      - `third_party/externals/sdl/src/atomic/SDL_spinlock.c`
      - `third_party/externals/skcms/src/Transform.c`
  - `ninja skqp`
    - note that many tests do not build in this config because we disable pdf
      and skottie
  - `./out/skqp --gtest_filter="SkiaGM_vk.*" platform_tools/android/apps/skqp/src/main/assets report`
- distribute
  - `mkdir skqp-dist`
  - `aarch64-linux-gnu-strip -xo skqp-dist/skqp out/skqp`
  - `ln -sf ../platform_tools/android/apps/skqp/src/main/assets skqp-dist/`
  - `tar zchf skqp-dist.tar.gz skqp-dist`
  - `rm -rf skqp-dist`
- old
  - `./out/skqp platform_tools/android/apps/skqp/src/main/assets skqp/rendertests.txt results`
    - `./out/skqp NOT USED results '^vk_arithmode$'` seems better

## Android

- skia on android
  - <https://android.googlesource.com/platform/external/skia> for platform
  - <https://android.googlesource.com/platform/external/skqp> for cts
  - <https://skia.org/docs/user/build/#android>
  - <https://skia.googlesource.com/skia/+/main/tools/skqp/README.md>
- to build an apk from skqp's `android12-tests-release` or
  `android11-tests-release` branch
  - install python2 and openjdk 8
    - `python-is-python2`
    - `openjdk-8-jre` and `openjdk-8-jdk`
  - install `ndk;16.1.4479499` and `platforms;android-26`
  - edit `DEPS` and `BUILD.gn` to remove all mentions of `Nima`
  - `tools/git-sync-deps`
  - `rm -rf platform_tools/android/apps/arcore`
  - comment out duplicated defines from `include/config/SkUserConfig.h`
  - `ANDROID_HOME=~/android/sdk ANDROID_NDK=~/android/sdk/ndk/16.1.4479499
    tools/skqp/make_universal_apk arm64`
- to build an apk from skqp's `pie-cts-release` branch
  - same as above
  - no need to remove Nima-Cpp and arcore
- note that `make_universal_apk` downloads models and sets up `resources`
  symlink
  - `tools/skqp/download_model`
  - `resources` symlink
- manual run
  - `adb install -r out/skqp/skqp-universal-debug.apk`
  - `adb shell am instrument -w org.skia.skqp`
  - `adb logcat org.skia.skqp skia "*:S"`
- CTS run
  - `./cts-tradefed run commandAndExit cts-dev -m CtsSkQPTestCases
       --module-arg 'CtsSkQPTestCases:include-filter:org.skia.skqp.SkQPRunner#gles_imageblur*'`
