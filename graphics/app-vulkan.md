Vulkan Loader
=============

## Build

- loader
  - `git clone https://github.com/KhronosGroup/Vulkan-Loader.git`
  - `cd Vulkan-Loader; mkdir out; cd out`
  - `../scripts/update_deps.py --generator Ninja`
    - this clones and builds `Vulkan-Headers`
  - `cmake -G Ninja -C helper.cmake ..`
  - `ninja`
- validation layers
  - `git clone https://github.com/KhronosGroup/Vulkan-ValidationLayers.git`
  - `cd Vulkan-ValidationLayers; mkdir out; cd out`
  - `../scripts/update_deps.py --generator Ninja`
    - this clones and builds `glslang`, `Vulkan-Headers`, `SPIRV-Headers`, and
      `robin-hood-hashing`
  - `cmake -G Ninja -C helper.cmake ..`
  - `ninja`
- tools
  - `git clone https://github.com/KhronosGroup/Vulkan-Tools.git`
  - `cd Vulkan-Tools; mkdir out; cd out`
  - `../scripts/update_deps.py --generator Ninja`
    - this clones and builds `Vulkan-Headers` and `Vulkan-Loader`
  - `cmake -G Ninja -C helper.cmake ..`
  - `ninja`
- LunarG layers
  - `git clone --recurse-submodules https://github.com/LunarG/VulkanTools.git`
  - `cd VulkanTools`
  - `./update_external_sources.sh`
  - `mkdir out; cd out`
  - `../scripts/update_deps.py --generator Ninja`
    - this clones and builds `Vulkan-Headers`, `Vulkan-Loader`,
      `Vulkan-Tools`, and `Vulkan-ValidationLayers`
  - `cmake -G Ninja -C helper.cmake ..`
  - `ninja`
- gfxreconstruct
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
- cross-compile hell
  - general
    - use `scripts/update_deps.py --no-build` to check out the right versions
    - `cmake -S. -Bout -DCMAKE_TOOLCHAIN_FILE=toolchain.mk`
      - `set(CMAKE_STAGING_PREFIX /tmp/hell)`
    - `make -C out install`
  - build `Vulkan-Headers`
    - nothing special
  - build `Vulkan-Loader`
    - `-DVULKAN_HEADERS_INSTALL_DIR=/tmp/hell`
    - `-DBUILD_WSI_XCB_SUPPORT=no`
    - `-DBUILD_WSI_XLIB_SUPPORT=no`
  - build `SPIRV-Headers`
    - nothing special
  - build `SPIRV-Tools`
    - `python utils/git-sync-deps`
  - build `robin-hood-hashing`
    - `-DRH_STANDALONE_PROJECT=off`
  - build `Vulkan-ValidationLayers`
    - `-DVULKAN_HEADERS_INSTALL_DIR=/tmp/hell`
    - `-DSPIRV_HEADERS_INSTALL_DIR=/tmp/hell`
    - `-DSPIRV_TOOLS_INSTALL_DIR=/tmp/hell`
    - `-DCMAKE_PREFIX_PATH=/tmp/hell/lib/cmake`
      - to find robin-hood-hasing
    - `-DBUILD_WSI_XCB_SUPPORT=off`
    - `-DBUILD_WSI_XLIB_SUPPORT=off`
    - `-DBUILD_LAYER_SUPPORT_FILES=on`
  - build `VulkanTools`
    - `set(CMAKE_FIND_ROOT_PATH ${CMAKE_SOURCE_DIR}/submodules)`
      - for jsoncpp
    - `set(CMAKE_CXX_FLAGS -Wno-shift-op-parentheses)`
    - comment out monitor layer in `layersvt/CMakeLists.txt`
    - `-DVULKAN_HEADERS_INSTALL_DIR=/tmp/hell`
    - `-DVULKAN_LOADER_INSTALL_DIR=/tmp/hell`
    - `-DVULKAN_VALIDATIONLAYERS_INSTALL_DIR=/tmp/hell`
    - `-DBUILD_WSI_XCB_SUPPORT=off`
    - `-DBUILD_WSI_XLIB_SUPPORT=off`

## Android

- <https://developer.android.com/ndk/guides/graphics/validation-layer>
  - apps can include the validation layer in their apks
  - On Android 9+, if an app is debuggable or if the system is userdebug with
    root, it can load an external layer
    - `adb push <layer.so> /data/local/tmp`
    - `adb shell run-as <com.example.app> cp /data/local/tmp/<layer.so> .`
    - `adb shell run-as <com.example.app> ls <layer.so>`
    - `adb shell setprop debug.vulkan.layers <layer>`
    - note, this does NOT work for me.  The loader does not search under
      `/data/data/com.example.app`
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
- gfxreconstruct android
  - to build,
    - `cd android`
    - `ANDROID_SDK_ROOT=<path-to-sdk-top-dir> sh gradlew assembleDebug`
  - to install,
    - `adb install -g -t -r
      ./tools/replay/build/outputs/apk/debug/replay-debug.apk`
    - `adb shell mkdir -p /data/local/debug/vulkan`
    - `adb push <path-to>/libVkLayer_gfxreconstruct.so /data/local/debug/vulkan`
  - to trace,
    - `adb shell setprop debug.vulkan.layers VK_LAYER_LUNARG_gfxreconstruct`
      - the loader will always implicitly load the layer
    - permissions
      - `adb shell pm grant com.name.app android.permission.READ_EXTERNAL_STORAGE`
      - `adb shell pm grant com.name.app android.permission.WRITE_EXTERNAL_STORAGE`
      - this does NOT work
        - i have to use `/data/local/debug/blah.gfxr` I created
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
- `Vulkan-Tools`
  - `cd build-android`
  - `ANDROID_SDK_HOME=<sdk-top> ANDROID_NDK_HOME=<ndk-top>
    PATH=$PATH:<ndk-top> ./build_all.sh`
  - `adb install ../cube/android/cube/bin/vkcube.apk`
  - `adb shell am start -W -S \
     -n com.example.VkCube/android.app.NativeActivity \
     -a android.intent.action.MAIN \
     -c android.intent.category.LAUNCHER \
     --es args '"--present_mode 2"'`

## Call Chain

- `vkCreateBuffer` in loader
  - implemented in `loader/trampoline.c`
  - get the dispatch table `VkLayerDispatchTable`
  - `CreateBuffer` points to icd's implementation or the first layer's
    implementation
- `vulkan_layer_chassis::CreateBuffer` in validation layers
  - implemented in `layers/generated/chassis.cpp`
  - for each validation object, call its `PreCallValidateCreateBuffer`
    - `ThreadSafety`, `StatelessValidation`, `ObjectLifetimes`, `CoreChecks`
  - for each validation object, call its `PreCallRecordCreateBuffer`
    - none
  - `DispatchCreateBuffer` calls into the driver (next layer?)
  - for each validation object, call its `PostCallRecordCreateBuffer`
    - `ThreadSafety`, `ObjectLifetimes`, `ValidationStateTracker`

## Validation Layers

- `ValidationObject`s such as `StatelessValidation` are created in
  `vulkan_layer_chassis::CreateInstance` and
  `vulkan_layer_chassis::CreateDevice`
- I guess `ValidationObject`s are set up in a way that the standard chassis
  code can be in charge of interception, etc.
- `StatelessValidation`
  - validate command parameters without any internal state
- `ObjectLifetimes`
  - very simple object lifetime validation
  - e.g., does not check if an in-flight VkFence is destroyed
- `CoreChecks`
  - all expensive stuff

## Four Classes of Functions

- Non-Dispatchable Functions
  - discovered through `vkGetInstanceProcAddr` without an instance
  - `vkEnumerateInstanceExtensionProperties`
  - `vkEnumerateInstanceLayerProperties`
  - `vkCreateInstance`
- Instance Functions
  - discovered through `vkGetInstanceProcAddr` with a valid instance
  - those whose first parameter is `VkInstance`
    - `vkDestroyInstance`
    - `vkEnumeratePhysicalDevices`
    - `vkGetInstanceProcAddr`
      - It is also discoverable through platform-specific means
    - `vkCreate*SurfaceKHR`
    - `vkDestroySurfaceKHR`
    - `vkCreateDebugReportCallbackEXT`
    - `vkDestroyDebugReportCallbackEXT`
    - `vkDebugReportMessageEXT`
- Physical Device Functions
  - discovered through `vkGetInstanceProcAddr` with a valid instance
  - those whose first parameter is `VkPhysicalDevice`
- Device Functions
  - discovered through `vkGetInstanceProcAddr` with a valid instance
  - can also be discovered through `vkGetDeviceProcAddr` with a valid device
  - those whose first parameter is one of `VkDevice`, `VkQueue`, and
    `VkCommandBuffer`

## Simplest Platforms: no layer library, single ICD

- A loader is not necessarily needed
  - it may still be desirable, to allow apps to always link to it and consume
    minimal resource unless used
- Such loader
  - exports `vkGetInstanceProcAddr`, which loads the ICD and forward all calls
    to the ICD
  - can be bypassed completely if an app looks up `vkGetInstanceProcAddr` using
    loader's `vkGetInstanceProcAddr'

## Platforms with no layer library but multiple ICDs

- Other being conformant, ICDs must make their `vkGetInstanceProcAddr`
  discoverable and allow the loader to embed a pointer in all dispatchable
  objects
- loader must implement all functions
  - clearly it must implement `vkGetInstanceProcAddr` and all
    non-dispatachable functions
  - since `vkGetInstanceProcAddr` can be used to query any function, it must
    return something without knowing which ICD to forward to.  As such, it
    must implement all functions.
  - when there is only one ICD, `vkGetInstanceProcAddr` knows where to
    forward calls to.  But if all functions are exported to apps, it could
    not make short cuts because those functions jump via dispatch tables
    and it must always set them up
- loader non-dispatchable functions
  - `vkEnumerateInstanceLayerProperties`
    - load all ICDs, call into them, and aggregate the results
    - unclear if there can be duplicated layer names
    - what if different ICDs return different layers?  Intersect instead of
      aggregate?
  - `vkEnumerateInstanceExtensionProperties`
    - load all ICDs
    - if used to enumerate a layer, forward to the first ICD supporting the
      layer
    - otherwise, call into all of them and aggregate
      - what if different ICDs return different extensions?  Intersect instead
        of aggregate?
      - since we must return a wrapped instance, must filter out unknown or
        unsafe instance extensions
  - `vkCreateInstance`
    - load all ICDs, call into all of them and aggregate.
    - A wrapped instance is returned.  All instance functions must go through
      the loader to unwrap.  Instance extensions that add new intance
      functions are disallowed.
    - dispatch table?  no, a `VkInstance` does not need a dispatch table
- instance functions
  - `vkCreateXxxSurfaceKHR`
    - call into all ICDs to aggregate
    - return a wrapped surface
  - `vkGetInstanceProcAddr`
    - return loader's versions for all classes of functions
    - if an unknown function is queried, must fail
      - it may be temping to generate a stub dynamically.  But how does the
        stub know how to unwrap and dispatch?
      - we filter out all unknown/unsafe instance extensions so this should
        never happen
  - `vkEnumeratePhysicalDevices`
    - call into all ICDs to aggregate
    - initialize physical device dispatch tables for all `VkPhysicalDevice`
      - memory for physical device dispatch tables is owned by instance
      - initialize to the results of ICDs' `vkGetInstanceProcAddr` calls
- loader physical device functions and device functions
  - get dispatch table and jump, except for those who return dispatchable
    objects and `vkGetDeviceProcAddr`
  - `vkGetDeviceQueue` and `vkAllocateCommandBuffers`
     - they return dispatchable objects
     - set up device dispatch tables additionally after dispatching
       - otherwise, "dispatch table and jump" will crash
  - `vkCreateDevice`
     - device dispatch table needs to be set up additionally
     - memory for device dispatch table is owned by device
     - initialize to the results of device's `vkGetDeviceProcAddr` calls
  - `vkGetDeviceProcAddr`
    - return loader's versions for `vkGetDeviceQueue`,
      `vkAllocateCommandBuffers` and itself
    - looks up in the device dispatch table then
    - for unknown functions (defined by extensions), call into the ICD

## Platforms with layer libraries and single ICD

- loader non-dispatchable functions
  - `vkEnumerateInstanceLayerProperties`
    - load all layer libraries and ICD, call into them, and aggregate the
      results
  - `vkEnumerateInstanceExtensionProperties`
    - load all layer libraries and ICD
    - if used to enumerate a layer, forward to the first layer library that
      supports the layer
    - otherwise, forward to ICD
  - `vkCreateInstance`
    - load all layer libraries (if any enabled) and ICD
    - add a chain of `vkGetInstanceProcAddr` for each enabled layer and ICD to
      `pCreateInfo->pNext`
    - call into the first layer/ICD
    - initialize instance dispatch table using first layer/ICD's
      `vkGetInstanceProcAddr`
- loader instance functions
  - `vkGetInstanceProcAddr`
    - return loader's version for non-dispatchable functions
    - otherwise, return from the instance dispatch table
    - otherwise, they are unknown.  Call into first layer/ICD.
  - `vkEnumeratePhysicalDevices`
    - call into first layer/ICD
    - initialize physical device dispatch tables for all `VkPhysicalDevice`
      - memory for physical device dispatch tables is owned by instance
      - initialize to the results of first layer/ICD's `vkGetInstanceProcAddr`
        calls
  - `vkCreateXxxSurfaceKHR` and others
    - get dispatch table and jump
- loader physical device functions and device functions
  - get dispatch table and jump, except for those who return dispatchable
    objects or `vkGetDeviceProcAddr` or `vkEnumerateDeviceLayerProperties`
  - `vkGetDeviceQueue` and `vkAllocateCommandBuffers`
     - they return dispatchable objects
     - set up device dispatch tables additionally after dispatching
       - otherwise, "dispatch table and jump" will crash
  - `vkCreateDevice`
     - set up chaining
     - device dispatch table needs to be set up additionally
     - memory for device dispatch table is owned by device
     - initialize to the results of device's `vkGetDeviceProcAddr` calls
  - `vkGetDeviceProcAddr`
    - return loader's versions for `vkGetDeviceQueue`,
      `vkAllocateCommandBuffers`, itself, and
      `vkEnumerateDeviceLayerProperties`
    - looks up in the device dispatch table then
    - for unknown functions (defined by extensions), call into the first
      layer/ICD
  - `vkEnumerateDeviceLayerProperties`
    - need to add additional layer libaries

## Layer Chaining

- Restrictions
  - These four functions are not chained
    - `vkGetInstanceProcAddr`
    - `vkEnumerateInstanceExtensionProperties`
    - `vkEnumerateInstanceLayerProperties`
    - `vkCreateInstance`

## Loader-Layer interface

- We want a layer library to be able to provide any number of instance/deviec
  layers
- Say a layer library supports instance layer L1 and L2.
  - prefer l1GetInstanceProcAddr and l1GetInstanceProcAddr to know their
    entrypoints respectively
  - otherwise, vkGetInstanceProcAddr needs to dispatch internally

- instance layers must export `vkEnumerateInstanceLayerProperties` and
  `vkEnumerateInstanceExtensionProperties`
- device layers must export `vkEnumerateDeviceLayerProperties` and
  `vkEnumerateDeviceExtensionProperties`
