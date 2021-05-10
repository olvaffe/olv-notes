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

* instance layers must export `vkEnumerateInstanceLayerProperties` and
  `vkEnumerateInstanceExtensionProperties`
* device layers must export `vkEnumerateDeviceLayerProperties` and
  `vkEnumerateDeviceExtensionProperties`
