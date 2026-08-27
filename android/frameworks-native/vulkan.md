# Android vulkan

## Initialization

- `EnsureInitialized` is called on demand to load driver and layers
- `driver::OpenHAL`
  - if updated driver is used, `UnloadBuiltinDriver` unloads built-in driver
    and `LoadUpdatedDriver` loads updated driver
  - otherwise, `LoadBuiltinDriver` loads built-in driver
    - it picks sphal or apex namespace depending on `ro.vulkan.apex`
    - `ro.hardware.vulkan` or `ro.board.platform` decides `vulkan.<prop>.so`
- `DiscoverLayers`
  - layer library paths are collected to `g_layer_libraries` and layer names
    are enumerated to `g_instance_layers`
    - if debuggable, `/data/local/debug/vulkan/libVkLayer*.so`
    - `<app>/libVkLayer*.so`
- `vkCreateInstance` calls `LayerChain::CreateInstance`
  - `OverrideLayerNames::Parse` collects enabled layer names
    - `GetLayersFromSettings` collects from settings
    - if none, `ParseDebugVulkanLayers` collects from `debug.vulkan.layers`
  - `ActivateLayers` sets up layers

## VVL

- steps
  - `adb root`
  - `adb shell setenforce 0`
  - download latest release from
    <https://github.com/KhronosGroup/Vulkan-ValidationLayers/releases>
  - `adb shell mkdir -p /data/local/debug/vulkan`
  - `adb push libVkLayer_khronos_validation.so /data/local/debug/vulkan`
  - `adb shell setprop debug.vulkan.layers VK_LAYER_KHRONOS_validation`
- when an app starts, there should be
  - `vulkan  : searching for layers in '/data/local/debug/vulkan'`
    - `DiscoverLayersInPathList` during init
  - `vulkan  : added global layer 'VK_LAYER_KHRONOS_validation' from library '/data/local/debug/vulkan/libVkLayer_khronos_validation.so'`
    - `AddLayerLibrary` during init
  - `vulkan  : Loaded layer VK_LAYER_KHRONOS_validation`
    - `ActivateLayers` during instance creation
- if `failed to open layer directory '/data/local/debug/vulkan': Permission denied`,
  - it is selinux

## Android

- <https://developer.android.com/ndk/guides/graphics/validation-layer>
  - apps can include the validation layer in their apks
  - On Android 9+, if an app is debuggable or if the system is userdebug with
    root, it can load an external layer
    - `adb push <layer.so> /data/local/tmp`
    - `adb shell run-as <com.example.app> cp /data/local/tmp/<layer.so> .`
    - `adb shell run-as <com.example.app> ls <layer.so>`
    - `adb shell setprop debug.vulkan.layers <layer>`, or
      - `adb shell settings put global enable_gpu_debug_layers 1`
      - `adb shell settings put global gpu_debug_app <com.example.app>`
      - `adb shell settings put global gpu_debug_layers <layer>`
    - note, this does NOT work for me.  The loader does not search under
      `/data/data/com.example.app`
      - it appears that the app must be debuggable
      - copy the layer to
        `/data/app/<package>-<hash>/lib/x86_64` works
  - On Android 10+, the app can additionally load an external layer from
    another apk
    - `adb shell settings put global enable_gpu_debug_layers 1`
    - `adb shell settings put global gpu_debug_app <com.example.app>`
    - `adb shell settings put global gpu_debug_layers <layer>`
    - `adb shell settings put global gpu_debug_layer_app <package>`
    - these settings persist reboots until explicitly deleted
    - `GraphicsEnvironment.java`
      - `setupGpuLayers`
        - it calls `debugLayerEnabled` to check both `ENABLE_GPU_DEBUG_LAYERS`
          and `GPU_DEBUG_APP`
        - if enabled for the app, it calls `setupGpuLayers` with
          `GPU_DEBUG_LAYERS`
      - `getDebugLayerPathsFromSettings`
        - if `debugLayerEnabled` returns true, return the library path from
          `GPU_DEBUG_LAYER_APP`
  - looking at the source code, the loader also searches
    `/data/local/debug/vulkan`
- validation layers android
  - `~/android/sdk/cmdline-tools/latest/bin/sdkmanager --install 'platforms;android-26'`
  - `cd build-android`
  - edit `jni/Application.mk` and `jni/shaderc/Application.mk`
    - set `APP_ABI` to the desired abis
    - set `APP_STL` to `c++_static`
  - `ANDROID_SDK_HOME=~/android/sdk ANDROID_NDK_HOME=~/android/sdk/ndk/25.1.8937393 PATH=~/android/sdk/ndk/25.1.8937393:$PATH ./build_all.sh`
  - `./install_all.sh` installs the apk for Android 10+
  - for Android 9,
    - `adb push bin/libs/lib/x86_64/libVkLayer_khronos_validation.so /data/local/tmp`
    - `adb shell run-as com.example.VkCube cp /data/local/tmp/libVkLayer_khronos_validation.so .`
    - `adb shell run-as com.example.VkCube ls`
    - `adb shell settings put global enable_gpu_debug_layers 1`
    - `adb shell settings put global gpu_debug_app com.example.VkCube`

## gfxreconstruct

- <https://github.com/LunarG/gfxreconstruct/blob/dev/HOWTO_android.md>
- steps
  - `adb root`
  - `adb shell setenforce 0`
  - download latest release from
    <https://github.com/LunarG/gfxreconstruct/releases>
  - `adb shell mkdir -p /data/local/debug/vulkan`
  - `adb push libVkLayer_gfxreconstruct.so /data/local/debug/vulkan`
  - `adb shell setprop debug.vulkan.layers VK_LAYER_LUNARG_gfxreconstruct`
  - `adb shell setprop debug.gfxrecon.capture_file /data/local/tmp/capture.gfxr`
    - use a path writable by app
- when an app starts, there should be
  - `vulkan  : searching for layers in '/data/local/debug/vulkan'`
  - `vulkan  : added global layer 'VK_LAYER_LUNARG_gfxreconstruct' from library '/data/local/debug/vulkan/libVkLayer_gfxreconstruct.so'`
  - `vulkan  : Loaded layer VK_LAYER_LUNARG_gfxreconstruct`
  - `gfxrecon: ...`

## API Dump

- one way to build the layer is to build angle with
  - `angle_enable_vulkan_api_dump_layer = true`
- steps
  - `adb push libVkLayer_lunarg_api_dump.so /data/local/debug/vulkan`
  - `adb shell setprop debug.vulkan.layers VK_LAYER_LUNARG_api_dump`
  - `adb shell setprop debug.vulkan.api_dump.log_filename /data/user/10/<package>/files/vk_apidump.txt`
