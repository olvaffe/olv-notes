GFXReconstruct
==============

## Build

- `git clone --recurse-submodules https://github.com/LunarG/gfxreconstruct.git`
- `cmake -S. -Bout -GNinja -DCMAKE_BUILD_TYPE=Debug`
- `ninja -C out`
- `ninja -C out install`, or
  - edit `VkLayer_gfxreconstruct.json` to point to local
    `libVkLayer_gfxreconstruct.so`
  - set `VK_LAYER_PATH` to where `VkLayer_gfxreconstruct.json` lives
  - `gfxrecon-capture.py -o <name.gfxr> <executable> ...` to capture
  - `gfxrecon-replay <name.gfxr>` to replay
    - might need `--wsi xlib` on xwayland

## Android Build

- to build,
  - if x86, add pre-compiled lz4
    - `git clone https://github.com/lz4/lz4.git`
    - create `jni/Android.mk` and `jni/Application.mk`
      - see `git show 27351d2978dfd4f2f934be6bbc4982bbc5099b9e`
    - `PATH=~/android/sdk/ndk/25.1.8937393:$PATH ndk-build`
  - `cd android`
  - edit `gradle/wrapper/gradle-wrapper.properties`
    - use a newer version (e.g., 7.6) if openjdk is too new (e.g., 17)
    - <https://docs.gradle.org/current/userguide/compatibility.html>
  - edit `build.gradle`
    - update the android gradle plugin version (e.g., 7.4.1)
    - <https://developer.android.com/studio/releases/gradle-plugin#updating-gradle>
  - edit `layer/build.gradle` and `tools/replay/build.gradle`
    - update `abiFilters`
- to install,
  - `adb install -g -t -r
    ./tools/replay/build/outputs/apk/debug/replay-debug.apk`
  - `adb shell mkdir -p /data/local/debug/vulkan`
  - `adb push <path-to>/libVkLayer_gfxreconstruct.so /data/local/debug/vulkan`
    - this does not work
    - use `/data/app/<package>-<hash>/lib/x86_64` instead
- to trace,
  - `adb shell setprop debug.vulkan.layers VK_LAYER_LUNARG_gfxreconstruct`
    - the loader will always implicitly load the layer
  - permissions
    - `adb shell pm grant com.name.app android.permission.READ_EXTERNAL_STORAGE`
    - `adb shell pm grant com.name.app android.permission.WRITE_EXTERNAL_STORAGE`
    - this requires app's manifest to request those permissions first
    - alternatively,
      - `adb shell setprop debug.gfxrecon.capture_file /data/data/<package>/abc.gfxr"`
  - settings
    - `adb push vk_layer_settings.txt /sdcard`
    - `adb shell setprop debug.gfxrecon.settings_path /sdcard/vk_layer_settings.txt`
    - or, specify them using props
      - `adb shell setprop debug.gfxrecon.capture_file /sdcard/blah.gfxr`
  - to debug,
    - look for `vulkan` or `gfxrecon` in logcat
- to replay,
  - `./scripts/gfxrecon.py replay /sdcard/<trace.gfxr>`
  - remember to unset `debug.vulkan.layers`

## GFXReconstruct

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
