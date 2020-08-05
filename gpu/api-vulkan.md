Vulkan
======

## Spec (1.2.149)

- 2. Fundamentals
  - 2.2. Execution Model
    - TODO
  - 2.3. Object Model
    - the handle to a dispatchable object is a pointer which is unique during
      the object's lifetime
    - the handle to a non-dispatchable object is an opaque `uint64_t`
      - it can be a pointer
      - it can be an index to a pool (potentially non-unique if pool objects
      	are reference counted)
      - it can be the object states tightly packed (thus non-unique)
  - 2.3.1. Object Lifetime
    - conceptually, an object has states and contents
      - states are immutable after object creation/allocation
      - contents can change
    - create/destroy has relatively low-frequency and alloc/free has
      relatively high-frequency
    - when an in-parameter is a pointer to app data, the ownership of the app
      data is temporarily transfered from the app to the driver.  That ends
      when the command returns.
    - clean up order
      - a VkInstance can be destroyed when all VkDevice are destroyed
      - a VkDevice device can be destroyed when all retrieved VkQueue are idle
      	and all children objects are destroyed
	- a pool object is the parent of its children objects; destroying the
	  pool objects implicitly frees all its children objects
      - when can a child object of a VkDevice be destroyed?
        - when it is not used by HW or by another object
    - command buffer lifecycle
      - initial, after alloc or reset
      - recording, after begin
      - executable, after end or execution
      - pending, after submission but before completion
      - invalid, after (one-time) execution or invalidated (e.g., by
      	destroying object that was recorded)
    - when a VkQueue is busy, these objects used by the queue must not be
      destroyed
      - IOW, these objects might own HW resources that must not be released
      - VkFence / VkSemaphore
      - VkCommandBuffer / VkCommandPool
    - when a command buffer is pending, meaning it is pending HW execution,
      these objects used by the command buffer must not be destroyed
      - IOW, these objects might own HW resources that must not be released
      - these probably own HW memory
        - VkBuffer / VkImage / VkDeviceMemory
        - VkDescriptorPool / VkDescriptorSet
        - VkCommandPool / VkCommandBuffer
        - VkPipeline
        - VkEvent
        - VkQueryPool
      - these probably own HW descriptors
        - VkBufferView / VkImageView / VkSampler / VkSamplerYcbcrConversion
        - VkRenderPass / VkFramebuffer
    - when a command buffer is recording or executable, destroying any object
      above used by the command buffer makes the command buffer invalid
      - IOW, the command buffer might have pointers to those objects and those
      	pointers become dangling
      - the command buffer can only be reset or freed when in the invalid
      	state
    - more generally, objects of a VkDevice can be destroyed in any order
      - with caveats, of course
      - when one objects uses another object
	- this other object might have been fully parsed, such as in
	  VkPipeline using VkShaderModule, and can be destroyed
	- the object might have a pointer to this other object, such as in
	  VkImageView using VkImage. It is allowed to destroy the other object
	  and create a dangling pointer, as along as the dangling pointer is
	  no longer dereferenced.  IOW, the object can only be reset or
	  destroyed.
    - these objects are consumed (e.g., fully parsed) when passed to a command
      that creates another object
      - they can be destroyed after the create command returns, without
      	worrying about dangling pointers
      - VkShaderModule and vkCreate*Pipeline
      - VkPipelineCache and vkCreate*Pipeline
      - VkRenderPass and vkCreate*Pipeline
      - VkRenderPass and vkCreateFramebuffer
      - VkDescriptorSetLayout and vkCreatePipelineLayout
        - not true for some impls
      - VkDescriptorSetLayout and vkCreateDescriptorUpdateTemplate
  - 2.3.2. External Object Handles
    - a VkDevice defines a object handle scope
      - any handle not in scope is said to be external
      - an external handle is exported from the object in the source scope
      - an external handle can be imported to an in-scope handle
  - 2.5. Command Syntax and Duration
    - VkBool32 can only be 0 or 1
    - commands that create/destroy objects take the form
      `vkCreate*`/`vkDestroy*`
      - either takes `VkAllocationCallbacks` as the last in-parameter
    - commands that alloc/free objects from a pool take the form
      `vkAllocate*`/`vkFree*`
      - they use the pool's `VkAllocationCallbacks`
    - `vkGet*` and `vkEnumerate*` results are invarant given the same
      in-parameters, unless called out
  - 2.6. Threading Behavior
    - an object is immutable, internally synced, or externally synced
      - immutable: no state change allowed thus no mutex needed
      - internally synced: an internal mutex protects some mutable states
      - externally synced: no two command use the same object at the same time
    - some commands affect children objects that are enumerated or allocated
      rather than created and the children objects are externally synced
      - VkInstance and VkPhysicalDevice
      - VkDevice and VkQueue
      - VkDescriptorPool and VkDescriptorSet
      - VkCommandPool and VkCommandBuffer
    - some object can either be internally or externally synced
      - `VK_PIPELINE_CACHE_CREATE_EXTERNALLY_SYNCHRONIZED_BIT_EXT`
  - 2.7.2. Implicit Valid Usage
    - `VK_NULL_HANDLE` is for non-dispatchable objects; `NULL` is for
      dispatchable objects
    - `NULL` is allowed for pointers unless called out
    - char pointer is non-NULL and terminated by a null character; when called
      out, it can be NULL
    - `_MAX_ENUM` is not a valid enumerant (the value can theoretically be
      valid however, if there are enough valid enumerants)
    - client must ignore bits in a flag that it does not understand; e.g.,
      driver might set bits returned by a instance command while the bits are
      gated by device extensions
    - sType always has a valid value
    - pNext is either NULL or a valid pointer
    - in a pnext chain, a struct must not appear more than once unless
      explicitlly called out
    - drivers must ignore unknown pnext node
      - it might be valid to a layer who cannot strip it out
    - some clarifications on core versons and extensions
  - 2.7.3. Return Codes
    - return code is either success (>=0) or runtime error (<0); no
      programming error
    - on errors, driver can trash out-parameters (except sType and pNext)
    - `VK_ERROR_UNKNOWN` is always a result of app bug or driver bug; the app
      should enable validation layers to get details, or to file a bug against
      the validation layers or the driver

## Versions

- if `vkGetInstanceProcAddr("vkEnumerateInstanceVersion")` returns NULL, the
  instance-level version is 1.0
- otherwise, `vkEnumerateInstanceVersion` returns instance-level version
- `VkPhysicalDeviceProperties::apiVersion` specifies the device-level version
- roughly, instance-level version and commands are decided by the loader;
  device-level version and commands are decided by the driver
  - a command taking none or VkInstance is a instance-level command
  - a command taking VkPhysicalDevice and others is a device-level command
- for instance-level version 1.0, `VkApplicationInfo::apiVersion` must be 1.0
  - it means the app can only use 1.0 for both instance-level and device-level
    functions
- since instance-level version 1.1, `VkApplicationInfo::apiVersion` can be any
  version
  - it specifies the max version of both instance-level and device-level
    versions the app intends to use
  - instance-level version can differ from device-level version

## Layers

- instance-level layers exist
- device-level layers are deprecated

## Extensions

- instance-level extensions
  - when an instance-level extension is not enabled, `vkGetInstanceProcAddr`
    for a command defined by the extension returns NULL
    - can a instance extension define a global command that are queriable with
      `vkGetInstanceProcAddr(NULL, ...)`?  It seems no.
  - when a device-level extension is availalbe, `vkGetInstanceProcAddr` for a
    command defined by the extension returns non-NULL
- device-level extensions
  - when a device-level extension is not enabled, `vkGetDeviceProcAddr` for a
    command defined by the extension returns NULL
  - physical-device-level commands defined by device extensions can be used as
    long as the device extensions are available

# Example

- create a window
  - `xcb_connect` to create a connection and find the root screen
  - `xcb_create_window` and `xcb_map_window`
- initialize vulkan
  - `vkEnumerateInstanceVersion`
  - `vkCreateInstance`
  - `vkEnumeratePhysicalDevices`
  - `vkGetPhysicalDeviceProperties2`
  - `vkGetPhysicalDeviceFeatures2`
  - `vkGetPhysicalDeviceMemoryProperties2`
  - `vkGetPhysicalDeviceQueueFamilyProperties2`
  - `vkGetPhysicalDeviceFormatProperties2`
  - `vkEnumerateDeviceExtensionProperties`
- initialize a logical device
  - `vkCreateDevice`
  - `vkGetDeviceQueue`
  - `vkCreateSemaphore` twice for presentComplete and renderComplete
- initialize a swapchain
  - `vkCreateXcbSurfaceKHR` from the xcb connection and window
  - `vkGetPhysicalDeviceSurfaceSupportKHR`
  - `vkGetPhysicalDeviceSurfaceFormatsKHR`
  - `vkCreateCommandPool`
  - `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`
  - `vkGetPhysicalDeviceSurfacePresentModesKHR`
  - `vkGetPhysicalDeviceSurfacePresentModesKHR`
  - `vkCreateSwapchainKHR`
  - `vkGetSwapchainImagesKHR`
  - `vkCreateImageView`
- 
  - `vkAllocateCommandBuffers`
  - `vkCreateFence`
- create depth buffer
  - `vkCreateImage`
  - `vkGetImageMemoryRequirements`
  - `vkAllocateMemory`
  - `vkBindImageMemory`
  - `vkCreateImageView`
- create render pass
  - `vkCreateRenderPass`
- create pipeline cache
  - `vkCreatePipelineCache`
- create framebuffer
  - `vkCreateFramebuffer`
- load textures
  - `vkCreateBuffer`
  - `vkGetBufferMemoryRequirements`
  - `vkAllocateMemory`
  - `vkBindBufferMemory`
  - `vkMapMemory`
  - `vkUnmapMemory`
  - `vkCreateImage`
  - `vkGetImageMemoryRequirements`
  - `vkAllocateMemory`
  - `vkBindImageMemory`
  - `vkCmdPipelineBarrier`
  - `vkCmdCopyBufferToImage`
  - `vkCmdPipelineBarrier`
  - `vkCreateSampler`
  - `vkCreateImageView`
- create VBO
  - `vkCreateBuffer` and ...
- create UBO
  - `vkCreateBuffer` and ...
- vertex descriptions
- descriptor set and pipeline layout
  - `vkCreateDescriptorSetLayout`
  - `vkCreatePipelineLayout`
- pipeline
  - `vkCreateGraphicsPipelines`
- descriptor pool and set
  - `vkCreateDescriptorPool`
  - `vkAllocateDescriptorSets`
  - `vkUpdateDescriptorSets`
- build command buffer
  - `vkBeginCommandBuffer`
  - `vkCmdBeginRenderPass`
  - `vkCmdSetViewport`
  - `vkCmdSetScissor`
  - `vkCmdBindDescriptorSets`
  - `vkCmdBindPipeline`
  - `vkCmdBindVertexBuffers`
  - `vkCmdBindIndexBuffer`
  - `vkCmdDrawIndexed`
  - (more commands for UI)
  - `vkCmdEndRenderPass`
  - `vkEndCommandBuffer`
- render loop
  - `xcb_poll_for_event`
  - `render`
    - `vkAcquireNextImageKHR`
    - `vkQueueSubmit`
    - `vkQueuePresentKHR`
    - `vkQueueWaitIdle`

# Objects

- VkInstance is an instance of Vulkan
- VkPhysicalDevice is used to query device caps and features
- VkDevice and the available VkQueue's are created together
- VkCommandBuffer are command buffers that get submitted to specific queues
- VkCommandPool is to enable suballocations of command buffers
- VkSemaphore is for cross-queue synchronization with submit granularity
- VkFence is for vk/cpu synchronization with submit granularity
  - wait is on the CPU side
- VkBuffer and VkImage are created without any backing store
- VkDeviceMemory, the backing store, is allocated separately
- VkBufferView and VkImageView are views of the pipelines into the buffers /
  images
- VkEvent is for vk/cpu synchronization with command granularity
  - wait is on the GPU side
- VkQueryPool is for querying GPU stats
- VkSampler describes a sampler (min filter, etc)
- VkShaderModule is a shader stage compiled from SPIR-V
- VkDescriptorSetLayout describes the layout (bindings) of a descriptor set
- VkPipelineLayout is a group of descriptor set layouts
- VkRenderPass describes a rendering pass abstractly; no image but only their
  formats, etc.
- VkFramebuffer is created from VkRenderPass and a list of VkImageView as the
  attachments
- VkPipelineCache is for faster pipeline creation
- VkPipeline is a huge object created from VkPipelineLayout and VkRenderPass
  and other states (but not VkFramebuffer)
- VkDescriptorPool is to enable suballocations of descriptor sets
- VkDescriptorSet describes shader resources

# Render Passes and Framebuffers

* A VkRenderPass contains a set of (abstract) attachments it will work on
  * each attachment is described with VkAttachmentDescription abstractly
  * format
  * load/store ops
  * initial layout: the layout the attachment is in when entering the render
    pass
  * final layout: the layout the attachment will be transitioned to by the
    implementation automatically when exiting the render pass
* A VkFramebuffer contains a set of (physical) attachments
  * each attachemnt is a VkImageView
  * VkFramebuffer is like a physical instance of a VkRenderPass
  * the separation is such that, when a pipeline is created for a render pass,
    the pipeline can work with any framebuffer compatible with the render pass.
* A subpass of a render pass works with a subset of the attachments
  * input attachemnts
  * color attachemnts
  * resolve attachemnts
  * depth attachemnt
  * preserve attachemnts
  * the implementation transitions all attachments to the specified layouts
    automatically when entering a subpass

# Descriptor Sets

- A VkDescriptorPool allocates enough memory (host or device, depending on
  the implementations) to hold the specified amount of descriptors of the
  specified types.
- A VkDescriptorSet grabs some of the descriptors from the pool according to
  its VkDescriptorLayout.
- A VkDescriptorSetLayout of a set has multiple bindings, with each binding
  corresponding to an array of descriptors of the same type.
- vkUpdateDescriptorSets writes the descriptors into the memory
- Types of descriptors
  - samplers
  - sampled images
  - storage images
  - uniform texel buffers
  - storage texel buffers
  - uniform buffers
  - storage buffers
  - input attachments (color attachments being read in following subpasses)
- A descriptor pool mallocs
  - `sizeof(descriptor_set) * maxSets`
  - `sizeof(descriptor) * numDescriptors`, for each descriptor type
- A descriptor set layout is described as
  - binding X: N descriptors of certain type used by certain shader stages
  - there can be arbitrarily many bindings, using arbitrarily binding numbers
  - this is enough for impl to calculate the total number of HW descriptors
    required and to maps "descriptor #n at binding X" to "HW descriptor #m"
- A descriptor set is suballocated from the pool
  - `vkUpdateDescriptorSets` is used to update a descriptor set
  - a descriptor set may simply hold shallow pointers to various VkImageView,
    VkSampler, VkBuffer, etc.  HW descriptors are generated at draw time.

# Pipelines

- A VkPipelineLayout consists of multiple descriptor set layouts and push
  constants.
  - Because a pipeline can work with multiple sets at the same time.
- A pipeline is created from a pipeline layout and many other states
  - renderPass / subpass
  - pipeline cache
  - base pipeline

# Command Buffers

- A command buffer consists of
  - a list of BOs, dynamically growing, with one chained to another
  - a vector of reloc entries (if resources may move on the impl)
  - a vector of binding table blocks
- A command pool is almost a dummy object to reset/free a group of command
  buffers in one go

# Execution and Memory Dependencies

- An operation is an arbitrary amount of work to be executed on the host, a
  device, or an external entity.
- An execution dependency is a guarantee that, for two sets of operations, the
  first set must happens-before the second set.
- An execution dependency chain is a sequence of execution dependencies, where
  for each pair of consecutive execution dependencies, there is at least one
  operation in the the second set of operations of the first execution
  dependency and in the first set of operations of the second execution
  dependency.
- Execution dependencies alone are not sufficient to guarantee that values
  resulting from writes in one set of operations can be read from another set
  of operations.
- Three additional types of operations are introduced to control memory
  access
  - Availability operation causes the values generated by the specified memory
    write accesses to become available to a memory domain for future access
    (e.g., flush the cache)
  - Memory domain operation causes values available to a source memory domain
    to become available to a destination memory domain.  (e.g., host domain,
    device domain, external domain, and foreign domain)
  - Visibility operation causes values available to a memory domain to become
    visible to the specified memory accesses (e.g., invalidate the cache)
- A memory dependency is an execution dependency which also includes
  availability and visibility operations.  It guarantees that
  - the first set of operations happens-before the availability operation
  - the availability operation happens-before the visibilitiy operation
  - the visibility operation happens-before the second set of operations.
- Some commands, such as a draw command, can consist of multiple operations
  known as pipeline stages.  They must adhere to the implicit ordering: a
  later pipeline stage must not happens-before an earlier pipeline stage.

# Synchronizations

- Synchronization commands introduce explicit execution dependencies or memory
  dependencies.
- A synchronization command has two synchronization scopes.  The first scope
  defines the types of operations before it that are considered to be in the
  first set of the execution dependency.  The second scope defines the types
  of operations after it that are considered to be in the second set of the
  execution dependency.
- It might have two memory access scopes when it also defines a memory
  dependency.  The first scope defines the types of memory accesses generated
  by the operations in the first set that are made available.  The second
  scope defines the types of memory accesses generated by the operations in
  the second set that the available values are visible by.
- when the first access scope includes `VK_ACCESS_HOST_WRITE_BIT`, the command
  includes a memory domain operation from the host domain to the device
  domain; when the second access scope include `VK_ACCESS_HOST_READ_BIT` or
  `VK_ACCESS_HOST_WRITE_BIT`, it includes a memory domain operation from the
  device domain to the host domain
- vkFlushMappedMemoryRegions includes an availability operation
- vkInvalidateMappedMemoryRanges includes an visibility operation
- vkQueueSubmit includes both a domain operation (from the host to the device)
  and a visibility operation

# Command Ordering

- Within a single queue, vkQueueSubmit are executed in call order
- Within a VkQueueSubmit, VkSubmitInfo are executed in array order
- Within a VkSubmitInfo, VkCommandBuffer are executed in array order
- Within a VkCommandBuffer, commands outside of render passes are executed in
  recorded order (this includes vkCmdBeginRenderPass and vkCmdEndRenderPass)
- Within a render pass subpass, commands are executed in recorded order
- Between render pass subpasses, there is no implicit ordering

# Resource Exclusive Ownership

- resources should only be accessed in the Vulkan instance that has exclusive
  ownership of the underlying memory
- only one Vulkan instance has exclusive ownership of a resource's underlying
  memory at any time
- it takes three steps to transfer exclusive ownership
  - the source instance or API releases exclusive ownership
  - wait with a fence or semaphore
  - the destination instance or API acquires exclusive ownership
- furthermore, within a Vulkan instance, resources created with
  `VK_SHARING_MODE_EXCLUSIVE` should only be accessed in the queue that has
  exclusive ownership
- releasing and acquiring exclusive ownership use `VkBufferMemoryBarrier` or
  `VkImageMemoryBarrier`

SPIR-V
======

# Tools

- SPIRV-Headers
  - <https://github.com/KhronosGroup/SPIRV-Headers>
  - `spirv.h` defines enums for opcodes, scopes, semantics, etc.
- SPIRV-Tools
  - <https://github.com/KhronosGroup/SPIRV-Tools>
  - depends on SPIRV-Headers
  - provides tools (assembler, disassembler, optimizer, linker, etc.) and
    `libSPIRV-Tools.a` for working with SPIR-V
- glslang
  - <https://github.com/KhronosGroup/glslang>
  - depends on SPIRV-Tools
  - provides `glslangValidator` to convert GLSL/HLSL to SPIR-V
- SPIRV-Cross
  - <https://github.com/KhronosGroup/SPIRV-Cross>
  - provides `spirv-cross` to convert SPIR-V to GLSL/HLSL/MSL
- shaderc
  - <https://github.com/google/shaderc>
  - depends on glslang and SPIRV-Tools
  - provides tool (`glslc`) and library (`libshaderc`) to convert GLSL/HLSL
    to SPIR-V
