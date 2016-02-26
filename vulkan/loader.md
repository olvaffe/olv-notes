Vulkan Loader
=============

## Four Classes of Functions

- Platform Functions
  - discovered through platform-specific mechanism
  - `vkGetInstanceProcAddr`
- Static Functions
  - discovered through `vkGetInstanceProcAddr` without an instance
  - `vkEnumerateInstanceExtensionProperties`
  - `vkEnumerateInstanceLayerProperties`
  - `vkCreateInstance`
- Instance Functions
  - discovered through `vkGetInstanceProcAddr` with a valid instance
  - those whose first parameter is one of `VkInstance` and `VkPhysicalDevice`
  - the spec is unclear whether `vkGetInstanceProcAddr` is also an instance
    function or not; I think yes.
- Device Functions
  - discovered through `vkGetInstanceProcAddr` with a valid instance
  - or discovered through `vkGetDeviceProcAddr` with a valid device
  - those whose first parameter is one of `VkDevice`, `VkQueue`, and
    `VkCommandBuffer`

## ICDs

- Other being conformant,  must make their `vkGetInstanceProcAddr`
  discoverable and allow the loader to embed a pointer in all dispatchable
  objects
- If a platform supports only a single ICD without layer libraries, there does
  not need to be a loader.
  - Or, the loader's `vkGetInstanceProcAddr` would load the ICD and forward
    all calls to the ICD
  - This allows `libvulkan.so` to be always linked without consuming much
    resource
  - It would be great if `vkGetInstanceProcAddr` can be used to discover
    `vkGetInstanceProcAddr`, bypassing the loader completely (spec unclear)
- If a platform supports multiple ICDs without layer libraries...
  - loader must implement all functions
    - clearly it must implement platform and static functions
    - since `vkGetInstanceProcAddr` can be used to query any function, it must
      return something without knowing which ICD to forward to.  As such, it
      must implement all functions.
      - when there is only one ICD, `vkGetInstanceProcAddr` knows where to
        forward calls to.  But if all functions are exported to apps, we could
        not make a short cut because those functions jump via a dispatch table
        and we must set that up
  - loader static functions
    - `vkEnumerateInstanceLayerProperties`
      - load all ICDs, call into them, and aggregate the results
      - unclear if there can be duplicated layer names
      - not all instance can support all layers this way; intersect instead of
        aggregate?
    - `vkEnumerateInstanceExtensionProperties`
      - load all ICDs
      - if a layer is provided, forward to the first ICD supporting the layer
      - otherwise, call into all of them and aggregate
      - not all instance can support all extensions this way; intersect instead
        of aggregate?
    - `vkCreateInstance`
      - load all ICDs, call into all of them and aggregate
  - loader non-physical-device instance functions
    - `vkGetInstanceProcAddr`
      - if no valid instance, return its versions of static functions
      - otherwise, dispatchkkkk
    - `vkEnumeratePhysicalDevices`
      - call into all ICDs to aggregate
      - initialize physical device dispatch table for all `VkPhysicalDevice`
        - memory owned by instance
        - for most, can initialize with ICDs' `vkGetInstanceProcAddr`
        - for `vkCreateDevice`, device dispatch table needs to be set up
          additionally.  Initialize the slot to loader's version which
          initialize device dispatch table as follows
          - for most, use device's `vkGetDeviceProcAddr` to initialize
          - for `vkGetDeviceQueue` and `vkAllocateCommandBuffers`, which return
            dispatchable objects, use loader's to set up device dispatch tables
            additionally
            - otherwise, "dispatch table and jump" will crash
          - for `vkGetDeviceProcAddr`, do not let `vkGetDeviceQueue`,
            `vkAllocateCommandBuffers` and itself be bypassed
    - `vkCreateXxxSurfaceKHR`
      - call into all ICDs to aggregate
  - loader physical device functions
    - get dispatch table and jump
  - loader device functions
    - get dispatch table and jump
- If a platform supports only a single ICD with layer libraries....
  - TODO
  - TODO
  - TODO
  - TODO

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
