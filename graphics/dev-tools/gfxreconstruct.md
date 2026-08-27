# GFXReconstruct

## Build

- `apt install liblz4-dev zlib1g-dev libxcb-glx0-dev libxcb-keysyms1-dev`
- `git clone --recurse-submodules https://github.com/LunarG/gfxreconstruct.git`
- `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug`
- `ninja -C out`
- `ninja -C out install`, or
  - edit `VkLayer_gfxreconstruct.json` to point to local
    `libVkLayer_gfxreconstruct.so`
  - set `VK_LAYER_PATH` to where `VkLayer_gfxreconstruct.json` lives
- dist
  - `mkdir gfxrecon`
  - `for i in $(find out/tools -type f -name 'gfxrecon*') out/layer/{libVkLayer_gfxreconstruct.so,VkLayer_gfxreconstruct.json}; do ln -sf ../$i gfxrecon/$(basename $i); done`
  - `strip gfxrecon/*`
  - `tar --zstd -hcf gfxrecon.tar.zst gfxrecon`
- headless-only build
  - `-DGFXRECON_ENABLE_OPENXR=OFF -DBUILD_WSI_XLIB_SUPPORT=OFF -DBUILD_WSI_XCB_SUPPORT=OFF -DBUILD_WSI_WAYLAND_SUPPORT=OFF`
- cross-compile
  - prep chroot with the same deps
  - use `CMAKE_TOOLCHAIN_FILE`
    - `set(CMAKE_FIND_ROOT_PATH "${CMAKE_CURRENT_SOURCE_DIR}/external/nlohmann-json")`
- headless-only ndk
  - `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug \
       -DLZ4_OPTIONAL=ON  -DGFXRECON_ENABLE_OPENXR=OFF \
       -DCMAKE_FIND_ROOT_PATH=$PWD/external/nlohmann-json \
       -DCMAKE_TOOLCHAIN_FILE=${ANDROID_NDK}/build/cmake/android.toolchain.cmake \
       -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-35 -DANDROID_CCACHE=ccache`
  - compile errors: comment out `android_native_app_glue.h`-related stuff

## Usage

- `export PATH=<gfxrecon>:$PATH`
- `export VK_LAYER_PATH=<gfxrecon>`
- `gfxrecon.py capture-vulkan -o <name.gfxr> <executable> ...` to capture
  - it runs `gfxrecon-capture-vulkan.py` script to set up envvars and run the
    executable
  - these envvars are always set
    - `VK_INSTANCE_LAYERS=VK_LAYER_LUNARG_gfxreconstruct`
    - `GFXRECON_CAPTURE_FILE=gfxrecon_capture.gfxr`
  - these envvars are from cmdline options
    - `VK_LAYER_PATH`
    - `GFXRECON_CAPTURE_FRAMES`
    - `GFXRECON_CAPTURE_FILE_TIMESTAMP`
    - `GFXRECON_CAPTURE_TRIGGER`
    - `GFXRECON_CAPTURE_TRIGGER_FRAMES`
    - `GFXRECON_CAPTURE_COMPRESSION_TYPE`
    - `GFXRECON_CAPTURE_FILE_FLUSH`
    - `GFXRECON_LOG_LEVEL`
    - `GFXRECON_LOG_FILE`
    - `GFXRECON_LOG_OUTPUT_TO_OS_DEBUG_STRING`
    - `GFXRECON_MEMORY_TRACKING_MODE`
- `gfxrecon.py replay <name.gfxr>` to replay
  - it runs `gfxrecon-replay` native binary
  - might need `--wsi xlib` on xwayland

## Android Build

- to build,
  - `cd android`
  - `./gradlew :replay:assembleDebug -Parm64-v8a`
    - we limit to `:replay` project defined in `settings.gradle`
    - wt set `arm64-v8a` property which maps to `abiFilters`
- to install
  - `adb install -r --force-queryable ./tools/replay/build/outputs/apk/debug/replay-debug.apk`
- to trace
  - as non-root, enable the layer for the specific app
    - there are 4 settings to set
    - the app must be debuggable
  - or, as root, enable the layer globally
  - useful settings
    - `debug.gfxrecon.capture_file` defaults to `/sdcard/gfxrecon_capture.gfxr`
      - the default requires the app to have `MANAGE_EXTERNAL_STORAGE`
      - good alternatives are
        - `/data/user/10/${AppName}/gfxrecon_capture.gfxr`
        - `/sdcard/Android/data/${AppName}/gfxrecon_capture.gfxr`
    - `debug.gfxrecon.capture_process_name`
      - capture only the specified app, useful when the layer is enabled globally
    - `debug.gfxrecon.capture_file_flush` defaults to `false`
      - flush packet after each write, useful when capturing a crashing app
- to replay,
  - `adb shell appops set --uid com.lunarg.gfxreconstruct.replay MANAGE_EXTERNAL_STORAGE allow`
  - `./scripts/gfxrecon.py replay /sdcard/<trace.gfxr>`
- to debug,
  - look for `vulkan` or `gfxrecon` in logcat

## Internals

- replay `vkCreateDevice`
  - `Application::Run`
  - `Application::PlaySingleFrame`
  - `FileProcessor::ProcessNextFrame`
  - `FileProcessor::ProcessBlocks`
  - `FileProcessor::ProcessFunctionCall`
  - `VulkanDecoder::Decode_vkCreateDevice`
  - `VulkanReplayConsumer::Process_vkCreateDevice`
  - `VulkanReplayConsumerBase::OverrideCreateDevice`
  - `vkCreateDevice`
  - ICD `CreateDevice`
- replay `vkCreateXcbSurfaceKHR`
  - `Application::Run`
  - `Application::PlaySingleFrame`
  - `FileProcessor::ProcessNextFrame`
  - `FileProcessor::ProcessBlocks`
  - `FileProcessor::ProcessFunctionCall`
  - `VulkanDecoder::Decode_vkCreateXcbSurfaceKHR`
  - `VulkanReplayConsumer::Process_vkCreateXcbSurfaceKHR`
  - `VulkanReplayConsumerBase::OverrideCreateXcbSurfaceKHR`
  - `VulkanReplayConsumerBase::CreateSurface`
  - `XcbWindow::CreateSurface`
  - ICD `vkCreateXcbSurfaceKHR`
  - if `--wsi xlib` is speficied, it can
    - `VulkanReplayConsumerBase::CreateSurface`
    - `XlibWindow::CreateSurface`
    - ICD `vkCreateXlibSurfaceKHR`
  - `--wsi wayland` uses the deprecated `wl_shell` and is not supported by
    modern compositors

## WSI

- replay `--wsi` maps
  - `auto`     to `WsiPlatform::kAuto`
  - `win32`    to `WsiPlatform::kWin32`
  - `xlib`     to `WsiPlatform::kXlib`
  - `xcb`      to `WsiPlatform::kXcb`
  - `wayland`  to `WsiPlatform::kWayland`
  - `metal`    to `WsiPlatform::kMetal`
  - `display`  to `WsiPlatform::kDisplay`
  - `headless` to `WsiPlatform::kHeadless`
- platform support
  - win supports win32 and headless
  - linux supports xlib, xcb, wayland, display, and headless
  - macos supports metal and headless
  - android only supports android and has no platform concept
