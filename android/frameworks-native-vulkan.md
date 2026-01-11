Android vulkan
==============

## `GraphicsEnv`

- activity main thread
  - `ActivityThread::setupGraphicsSupport` calls `GraphicsEnvironment::setup`
    - `setupGpuLayers`
      - `setLayerPaths_native` calls `GraphicsEnv::setLayerPaths` to add
        search paths (e.g., app dir for bundled layers)
      - if debuggable, and `enable_gpu_debug_layers` and `gpu_debug_app`
        settings are set, `setDebugLayers_native` calls
        `GraphicsEnv::setDebugLayers` and `setDebugLayersGLES_native` calls
        `GraphicsEnv::setDebugLayersGLES`
    - `setupAngle` choses native or angle gles driver
    - `chooseDriver` chooses built-in or updated driver
- `GraphicsEnv`
  - `setDriverPathAndSphalLibraries` is called when updated driver is used
    - `getDriverNamespace` returns null typically unless updated driver is used
  - `setDriverToLoad`, `setVulkanInstanceExtensions`, etc. are called from
    egl and vulkan loaders to update gpu stats
  - `setLayerPaths` and `setDebugLayers*` are called from java to set up app
    debugging layers, if any

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
