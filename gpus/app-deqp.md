dEQP
====

## Build and Run

- Build
  - `python3 external/fetch_sources.py`
  - `mkdir out`
  - `cd out`
  - `cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Debug -DDEQP_TARGET=surfaceless`
    - `DEQP_TARGET` is mostly for GL/GLES
  - `ninja`
- Run Vulkan
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

- Download SDK and NDK
  - Bootstrap to `~/android-sdk`
    - <https://developer.android.com/studio>
    - choose "Command line tools only"
    - `unzip commandlinetools-linux-*_latest.zip`
    - `./cmdline-tools/bin/sdkmanager --sdk_root=~/android-sdk cmdline-tools\;latest`
    - `rm -rf commandlinetools-linux-*_latest.zip cmdline-tools`
  - Install packages
    - `cd ~/android-sdk`
    - `./cmdline-tools/latest/bin/sdkmanager --list`
    - `./cmdline-tools/latest/bin/sdkmanager --install "build-tools;29.0.3" "ndk;23.1.7779620" "platforms;android-28"`
- Build
  - `git remote add aosp https://android.googlesource.com/platform/external/deqp`
  - `git fetch aosp`
  - `git checkout -b android-11r6 android-cts-11.0_r6`
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
- CTS
  - download and unzip CTS
  - `./android-cts/tools/cts-tradefed`
  - `run cts -m CtsDeqpTestCases -t dEQP-VK.ubo.random.scalar#62 -a x86_64`
  - logcat says `Writing test log into /sdcard/TestLog.qpa`

## Overview

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

## Vulkan CTS

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

## GLES CTS

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
