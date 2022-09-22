Skia
====

## Build and Use

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
    - `skia_use_egl = true` to pick EGL/GLES
    - `skia_use_x11 = false` to pick GLX/GL
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
  - misc
    - `cc_wrapper = "ccache"`
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
- skia on angle
  - <https://skia.org/docs/user/special/angle/>
- skia on vulkan
  - <https://skia.org/docs/user/special/vulkan/>
- skia on android
  - <https://skia.org/docs/user/build/#android>
  - <https://skia.googlesource.com/skia/+/main/tools/skqp/README.md>

## Android

- <https://android.googlesource.com/platform/external/skia> for platform
- <https://android.googlesource.com/platform/external/skqp> for cts
- `android11-tests-release`
  - install `python-is-python2`
  - args
    - enable `skia_enable_pdf`
  - `(cd include/config/ && ln -sf ../../linux/include/config/SkUserConfig.h)`
  - edit `include/config/SkUserConfigManual.h` and
    - comment out `SK_BUILD_FOR_ANDROID_FRAMEWORK`
    - `#define SK_SUPPORT_GPU 1`
- `pie-cts-release`
  - install `python-is-python2`
  - args
    - enable `skia_enable_pdf`
    - remove `skia_enable_fontmgr_android`
    - remove `skia_use_gl`
    - remove `skia_use_x11`
    - add `extra_cflags = [ "-Wno-error" ]`
  - edit `include/config/SkUserConfigManual.h` and
    - comment out `SK_BUILD_FOR_ANDROID_FRAMEWORK`
    - `#define SK_SUPPORT_GPU 1`
  - edit `include/config/SkUserConfig.h` and remove those defined by `-D`
    already
    - most defines need to be commented out
  - fix aarch64 compile issues by using the slow paths
    - `src/jumper/SkJumper_stages.cpp`
    - `third_party/externals/sdl/src/atomic/SDL_spinlock.c`
    - `third_party/externals/skcms/src/Transform.c`

## SkQP

- modifications
  - edit `tools/skqp/src/skqp.h` to set `fEnforcedAndroidAPILevel` to 99
    - otherwise, all tests are skipped
  - to see the list of tests, add `printf` to `get_unit_tests`
- `./out/skqp . report` to run all tests
  - `.` is the asset dir: skqp looks for `resources` under the asset dir
  - `report` is for test results
- tests
  - `DEF_GANESH_TEST_*`
    - defines a test of type `skiatest::TestType::kGanesh`
    - a gpu test
  - `DEF_TEST`
    - defines a test of type `skiatest::TestType::kCPU`
    - a gpu test
  - skqp only tests ganesh tests
- `android11-tests-release`
  - no need to modify `fEnforcedAndroidAPILevel`
  - add printf to `get_render_tests` and `get_unit_tests` to list all tests
  - `./out/skqp . report '^vk_'`
- `pie-cts-release`
  - no need to modify `fEnforcedAndroidAPILevel`
  - add printf to `register_skia_tests` to list all tests
  - `./out/skqp --gtest_filter="SkiaGM_vk.*" . report`
- old
  - `./out/skqp platform_tools/android/apps/skqp/src/main/assets skqp/rendertests.txt results`
    - `./out/skqp NOT USED results '^vk_arithmode$'` seems better
