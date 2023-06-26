Vulkan
======

## History

- 1.0 was released in 2016
  - HW features are about GL 4.3 or GLES 3.1
  - plus persistent and coherent memory
- 1.1 was released in 2018
  - protected content
  - subgroup operations
  - multi-view
  - device groups
  - external memory/fence/semaphore
  - advanced compute
  - HLSL
  - YCbCr
- 1.2 was released in 2020
  - no new HW feature
  - timeline semaphore
  - (optional) descriptor indexing
- 1.3 was released in 2022
  - no new HW feature

## Chapter 3. Fundamentals

- 3.2. Execution Model
  - Work is submitted to queues using queue submission commands that
    typically take the form vkQueue* (e.g. vkQueueSubmit ,
    vkQueueBindSparse)
    - optionally take a list of semaphores upon which to wait before work
      begins and a list of semaphores to signal once work has completed
  - The work itself, as well as signaling and waiting on the semaphores are
    all queue operations
  - There are no implicit ordering constraints between queue operations on
    different queues, or between queues and the host, so these may operate
    in any order with respect to each other
    - Explicit ordering constraints between different queues or with the
      host can be expressed with semaphores and fences
  - queue operations on the same queue also may execute in any order, except
    when implicit ordering constraints apply
    - vkQueueSubmit respects submission order and other implicit ordering
      constraints
      - other queue submission commands do not
    - any queue operation cannot be reordered after a queue signal operation
      - any memory write also becomes available with queue signal operation
    - a queue wait operation cannot be reordered before a queue signal
      operation sharing the same semaphore
      - any memory write that is available also becomes visible
  - commands recorded in a command buffer are either
    - action command (draw, dispatch, clear, copy, begin/end renderpass)
    - state command (bind pipelines, set dynamic states, etc.)
    - sync command (set/wait events, barriers, render pass deps)
  - more in 7.2.
- 3.3. Object Model
  - the handle to a dispatchable object is a pointer which is unique during
    the object's lifetime
  - the handle to a non-dispatchable object is an opaque `uint64_t`
    - it can be a pointer
    - it can be an index to a pool (potentially non-unique if pool objects
      are reference counted)
    - it can be the object states tightly packed (thus non-unique)
  - 3.3.1. Object Lifetime
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
      - VkShaderModule and `vkCreate*Pipeline`
      - VkPipelineCache and `vkCreate*Pipeline`
      - VkRenderPass and `vkCreate*Pipeline`
      - VkRenderPass and vkCreateFramebuffer
      - VkDescriptorSetLayout and vkCreatePipelineLayout
        - not true for some impls
      - VkDescriptorSetLayout and vkCreateDescriptorUpdateTemplate
  - 3.3.2. External Object Handles
    - a VkDevice defines a object handle scope
      - any handle not in scope is said to be external
      - an external handle is exported from the object in the source scope
      - an external handle can be imported to an in-scope handle
- 3.5. Command Syntax and Duration
  - VkBool32 can only be 0 or 1
  - commands that create/destroy objects take the form
    `vkCreate*`/`vkDestroy*`
    - either takes `VkAllocationCallbacks` as the last in-parameter
  - commands that alloc/free objects from a pool take the form
    `vkAllocate*`/`vkFree*`
    - they use the pool's `VkAllocationCallbacks`
  - `vkGet*` and `vkEnumerate*` results are invarant given the same
    in-parameters, unless called out
- 3.6. Threading Behavior
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
- 3.7. Valid Usage
  - 3.7.2. Implicit Valid Usage
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
- 3.8. VkResult Return Codes
  - return code is either success (>=0) or runtime error (<0); no
    programming error
  - on errors, driver can trash out-parameters (except sType and pNext)
  - `VK_ERROR_UNKNOWN` is always a result of app bug or driver bug; the app
    should enable validation layers to get details, or to file a bug against
    the validation layers or the driver

## Chapter 6. Command Buffers

- secondary command buffer inherits no state from the primary command
  buffer
  - except for render pass and subpass
  - secondary command buffer must explicitly set all other states
- 6.4. Command Buffer Recording
  - `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`
    - an impl might generate self-modifying commands when the flag is set
      - how?
    - an impl might skip CPU-intensive optimizations when the flag it set
  - `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`
    - an impl might need to patch the last command of a secondary command
      buffer to jump back to the address of its caller
    - patching is not possible when the flag is set
- 6.7. Secondary Command Buffer Execution
  - vkCmdExecuteCommands can be inside or outside of a render pass
    - VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT must be set
      accordingly

## Chapter 7. Synchronization and Cache Control

- there are five explicit synchronization mechanisms
  - fences, semaphores, events, pipeline barriers, and render passes
- 7.1. Execution and Memory Dependencies
  - an operation is an arbitrary amount of work to be executed on
    - the host,
    - a device, or
    - an external entity such as presentation engine
  - a synchronization command introduces, between two sets of operations,
    - an execution dependency
      - the first synchronization scope defines the first set of operations
      - the second synchronization scope defines the second set of
        operations
    - a memory dependency
      - the first access scope defines the first set of memory accesses
      - the second access scope defines the second set of memory accesses
      - this is needed to deal with memory caches and domains
  - an execution dependency guarantees the first set of operations
    happens-before the second set of operations
    - i.e., the first set complete before the second set is initiated
    - not enough for RAW hazard, when the first set writes and the second
      set reads
    - nor enough for WAW hazard
  - types of memory access operations include
    - an availability operation makes writes available to a memory domain
      (e.g., vkFlushMappedMemoryRanges makes host writes available in host
      domain)
    - a memory domain operation makes writes available in one domain
      available in another (e.g., `VK_ACCESS_HOST_WRITE_BIT` in source
      access mask makes host writes available in device domain)
    - a visibility operation makes writes available become visible to the
      specified memory accesses (e.g., writes available in device domain are
      made visibile to accesses defined by the destination access mask)
  - a memory dependency is an execution dependency that includes availibitiy
    and visibility operations such that
    - the first set of operations (defined by the first sync scope)
      happens-before the availibility operation
    - the availibility operation happens before the visibility operation
      - the availibility operation applies to memory accesses that are
        defined the first access scope
      - the visibility operation applies to memory accesses that are
        defined the second access scope
      - note that the sync scopes and the access scopes are different, but
        the sync scopes often limit the access scopes
    - the visibility operation happens-before the second set of operations
      (defined by the second sync scope)
- 7.2. Implicit Synchronization Guarantees
  - command buffers and submission order
    - we need to give a meaning to the order in which commands are recorded
      and submitted
      - such that a draw command is not reordered before a state command it
        depends on
      - or two overlapping triangles are drawn in reverse order
        - this is known as the primitive order and the rasterization order
    - submission order essentially says that the commands must be executed
      in the order that they are recorded and submitted
      - except for commands in different subpasses of a render pass
    - however, execution can overlap or be out-of-order
      - as long as, for example, the primitive order is honored
  - explicit sync is required to avoid overlapping or reordering
    - e.g., non-primitive WAR, such as ssbo, requires an execution
      dependency
    - e.g., non-primitive RAW/WAW, such as ssbo, requires a memory
      dependency
  - semaphores and signal operation order
    - we need to give a meaning to the order in which queue signal
      operations are submitted
      - such that two queue signal operations are not reordered
    - signal operation order essentially says that the queue signal
      operations must be executed in the order that they are submitted
  - submission order and signal operation order are trivial
    - they exist only because queue operations are allowed to be reordered
      arbitrarily which is too wild; we need them as fundamental ordering
      contraints
    - they don't specify how a command buffer is ordered against its signal
      operation
      - that will be defined by the semaphore's execution dependency
    - they also don't specify memory dependency
      - say, we update an ssbo and sample from it; the submission order
        specifies the execution dependency but there is still a memory
        dependency that needs to be be specified
      - `vkCmdPipelineBarrier` to the rescue
- 7.4. Semaphores
  - 7.4.1. Semaphore Signaling
    - the semaphore signal operation is a synchronization command
      - it defines two synchronization scopes and their execution and memory
        dependencies
    - the first sync scope includes all commands in the batch
    - the second sync scope includes the signal operation itself
    - these are enough to make sure the semaphore signals after all commands
      in the batch are executed
      - and because of the fundamental signal operation order, all commands
        before the batch and all prior signal operations are also executed
    - IOW, a command can be reorderd before a prior signal operation and but
      after its own signal operation
  - 7.4.2. Semaphore Waiting
    - the first sync scope includes all semaphore signal operations that
      operate on the same semaphore
    - the second sync scope includes all commands in the batch
- 7.5. Events
  - events can be used for fine-grained host/queue synchronization
  - they can also be used as a "split" barrier
    - say, we update an ssbo and sample from it; there is an irrelevant work
      that we can insert before vkCmdPipelineBarrier to better utilize the
      GPU
    - when the irrelevant work is complex enough that might confuse
      vkCmdPipelineBarrier or make it impractical, we can instead
      vkCmdSetEvent after ssbo update and vkCmdWaitEvents before sampling
    - the barrier is splitted into the signal half and wait half

## Chapter 8. Render Pass

- 8.4. Render Pass Commands
  - depending on VkSubpassContents, any subpass of a render pass either
    execute only commands in the primary command buffer or only command the
    secondary command buffers

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

## Example

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

## Objects

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

## Render Passes and Framebuffers

- A VkRenderPass contains a set of (abstract) attachments it will work on
  - each attachment is described with VkAttachmentDescription abstractly
  - format
  - load/store ops
  - initial layout: the layout the attachment is in when entering the render
    pass
  - final layout: the layout the attachment will be transitioned to by the
    implementation automatically when exiting the render pass
- A VkFramebuffer contains a set of (physical) attachments
  - each attachemnt is a VkImageView
  - VkFramebuffer is like a physical instance of a VkRenderPass
  - the separation is such that, when a pipeline is created for a render pass,
    the pipeline can work with any framebuffer compatible with the render pass.
- A subpass of a render pass works with a subset of the attachments
  - input attachemnts
  - color attachemnts
  - resolve attachemnts
  - depth attachemnt
  - preserve attachemnts
  - the implementation transitions all attachments to the specified layouts
    automatically when entering a subpass

## Descriptor Sets

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

## Pipelines

- A VkPipelineLayout consists of multiple descriptor set layouts and push
  constants.
  - Because a pipeline can work with multiple sets at the same time.
- A pipeline is created from a pipeline layout and many other states
  - renderPass / subpass
  - pipeline cache
  - base pipeline

## Command Buffers

- A command buffer consists of
  - a list of BOs, dynamically growing, with one chained to another
  - a vector of reloc entries (if resources may move on the impl)
  - a vector of binding table blocks
- A command pool is almost a dummy object to reset/free a group of command
  buffers in one go

## Execution and Memory Dependencies

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

## Synchronizations

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

## Command Ordering

- Within a single queue, vkQueueSubmit are executed in call order
- Within a VkQueueSubmit, VkSubmitInfo are executed in array order
- Within a VkSubmitInfo, VkCommandBuffer are executed in array order
- Within a VkCommandBuffer, commands outside of render passes are executed in
  recorded order (this includes vkCmdBeginRenderPass and vkCmdEndRenderPass)
- Within a render pass subpass, commands are executed in recorded order
- Between render pass subpasses, there is no implicit ordering

## Resource Exclusive Ownership

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

## Debug Extensions

- `VK_EXT_debug_report`
  - deprecated by `VK_EXT_debug_utils`
- `VK_EXT_debug_marker`
  - promoted to `VK_EXT_debug_utils`
- `VK_EXT_validation_flags`
  - deprecated by `VK_EXT_validation_features`
- `VK_EXT_debug_utils`
- `VK_EXT_tooling_info`
  - implemented by tools and layers to report themselves
- `VK_EXT_validation_features`
  - enable/disable specific validation features of
    `VK_LAYER_KHRONOS_validation`
    - thread safety
    - api params
    - object lifetimes
    - core checks
    - unique handles

## `VkFormat`

- core formats
  - most core formats have one plane
  - compressed formats have one plane
  - depth formats have one plane
  - stencil formats have one plane
  - depth/stencil formats have two planes
    - `VK_FORMAT_D32_SFLOAT_S8_UINT`
    - `VK_FORMAT_D24_UNORM_S8_UINT`
- YCbCr formats
  - some have one plane, such as
    - `VK_FORMAT_G8B8G8R8_422_UNORM`
  - some have two planes, such as
    - `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM`
  - some have three planes, such as
    - `K_FORMAT_G8_B8_R8_3PLANE_422_UNORM`
- `VkImageTiling` and `VkImageLayout`
  - for each plane, HW needs to store the format data and often also format
    metadata
  - layout transition makes sure the format metadata is resolved to the format
    data for the destination operation
  - tiling specifies whether the format data is tiled or not
  - Image layout is per-image subresource.  Separate image subresources of the
    same image can be in different layouts at the same time, with the
    exception that depth and stencil aspects of a given image subresource can
    only be in different layouts if the `separateDepthStencilLayouts` feature
    is enabled.
- DRM modifiers
  - `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT` mixes tiling and layout
    together.  Format data and format metadata reside on different memory
    planes
  - a single-plane format thus can have multiple memory planes when the tiling is
    `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT`
  - can we combine multi-planar format with drm modifier?  It has `N*M`
    planes in theory.
- `VK_IMAGE_CREATE_DISJOINT_BIT`
  - driver by default allocates a buffer and puts planes on different region of
    the same buffer
    - thus only a `VkDeviceMemory` needs to be bound
  - when `VK_IMAGE_CREATE_DISJOINT_BIT` is specified, driver puts different
    planes on different buffers
    - thus multiple `VkDeviceMemory` need to be bound
  - as such, internally, a `VkImage` can point to multiple `VkDeviceMemory`
    which respectively point to different BOs

## `VkImageAspectFlagBits`

- basic
  - `VK_IMAGE_ASPECT_COLOR_BIT`
  - `VK_IMAGE_ASPECT_DEPTH_BIT`
  - `VK_IMAGE_ASPECT_STENCIL_BIT`
  - `VK_IMAGE_ASPECT_METADATA_BIT`
- multi-planar
  - `VK_IMAGE_ASPECT_PLANE_0_BIT`
  - `VK_IMAGE_ASPECT_PLANE_1_BIT`
  - `VK_IMAGE_ASPECT_PLANE_2_BIT`
- DRM modifier
  - `VK_IMAGE_ASPECT_MEMORY_PLANE_0_BIT_EXT`
  - `VK_IMAGE_ASPECT_MEMORY_PLANE_1_BIT_EXT`
  - `VK_IMAGE_ASPECT_MEMORY_PLANE_2_BIT_EXT`
  - `VK_IMAGE_ASPECT_MEMORY_PLANE_3_BIT_EXT`
- `vkGetImageMemoryRequirements2` and `vkGetDeviceImageMemoryRequirements`
  - aspect bit can be specified with `VkImagePlaneMemoryRequirementsInfo` or
    `VkDeviceImageMemoryRequirements`
  - only plane and memory plane aspects are valid
  - only when `VK_IMAGE_CREATE_DISJOINT_BIT` is set
  - only for drm modifiers or multi-plane formats
    - if drm modifiers, must be one of
      `VK_IMAGE_ASPECT_MEMORY_PLANE_0_BIT_EXT` and etc.
    - otherwise, if multi-plane formats, must be one of
      `VK_IMAGE_ASPECT_PLANE_0_BIT` and etc.
- `vkBindImageMemory2`
  - aspect bit can be specified with `VkBindImagePlaneMemoryInfo`
  - only plane and memory plane aspects are valid
- `vkGetImageSubresourceLayout` or `vkQueueBindSparse`
  - aspect mask can be specified with `VkImageSubresource`
  - must be a single bit
    - meaning depth/stencil formats have no well-defined linear layout
- `vkCmdCopyImage2`, `vkCmdCopyBufferToImage2`, `vkCmdCopyImageToBuffer2`,
  `vkCmdBlitImage2`, `vkCmdResolveImage2`
  - aspect mask can be specified with `VkImageSubresourceLayers`
  - only color, depth, stencil, and plane aspects are valid
- `vkCreateImageView`
  - aspect mask can be specified in `VkImageSubresourceRange`
  - valid aspects depend on formats
    - can only be `VK_IMAGE_ASPECT_COLOR_BIT` if color formats
    - can only be `VK_IMAGE_ASPECT_DEPTH_BIT` if depth-only formats
    - can only be `VK_IMAGE_ASPECT_STENCIL_BIT` if stencil-only formats
    - can only be depth, stencil, or both aspects if depth/stencil formats
      - must be a single bit when used in a descriptor set
      - is ignored and assume to be both aspects when used as a framebuffer
      	attachment
- `vkCmdClearColorImage`, `vkCmdClearDepthStencilImage`, `vkCmdSetEvent2`,
  `vkCmdWaitEvents2`, `vkCmdPipelineBarrier2`, `vkCmdWaitEvents`, and
  `vkCmdPipelineBarrier`
  - aspect mask can be specified in `VkImageSubresourceRange`
- `vkCreateRenderPass2` or `vkCreateRenderPass`
  - aspect mask can be specified in `VkAttachmentReference2` or
    `VkInputAttachmentAspectReference`
  - only for input attachments
- `vkCmdClearAttachments`
  - aspect mask can be specified in `VkClearAttachment`
  - valid aspect masks are color, depth, stencil, or depth+stencil

## `VkImageLayout`

- 1.0
  - `VK_IMAGE_LAYOUT_UNDEFINED`
  - `VK_IMAGE_LAYOUT_GENERAL`
  - `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
  - `VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL`
  - `VK_IMAGE_LAYOUT_DEPTH_STENCIL_READ_ONLY_OPTIMAL`
  - `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`
  - `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`
  - `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`
  - `VK_IMAGE_LAYOUT_PREINITIALIZED`
- 1.1
  - `VK_IMAGE_LAYOUT_DEPTH_READ_ONLY_STENCIL_ATTACHMENT_OPTIMAL`
  - `VK_IMAGE_LAYOUT_DEPTH_ATTACHMENT_STENCIL_READ_ONLY_OPTIMAL`
- 1.2
  - `VK_IMAGE_LAYOUT_DEPTH_ATTACHMENT_OPTIMAL`
  - `VK_IMAGE_LAYOUT_DEPTH_READ_ONLY_OPTIMAL`
  - `VK_IMAGE_LAYOUT_STENCIL_ATTACHMENT_OPTIMAL`
  - `VK_IMAGE_LAYOUT_STENCIL_READ_ONLY_OPTIMAL`
- 1.3
  - `VK_IMAGE_LAYOUT_READ_ONLY_OPTIMAL`
  - `VK_IMAGE_LAYOUT_ATTACHMENT_OPTIMAL`

## Depth and Stencil

- assume
  - `VK_KHR_maintenance2`
    - `VK_IMAGE_LAYOUT_DEPTH_ATTACHMENT_STENCIL_READ_ONLY_OPTIMAL_KHR`
    - `VK_IMAGE_LAYOUT_DEPTH_READ_ONLY_STENCIL_ATTACHMENT_OPTIMAL_KHR`
  - `VK_KHR_separate_depth_stencil_layouts`
    - `separateDepthStencilLayouts` feature
    - `VK_IMAGE_LAYOUT_DEPTH_ATTACHMENT_OPTIMAL_KHR`
    - `VK_IMAGE_LAYOUT_DEPTH_READ_ONLY_OPTIMAL_KHR`
    - `VK_IMAGE_LAYOUT_STENCIL_ATTACHMENT_OPTIMAL_KHR`
    - `VK_IMAGE_LAYOUT_STENCIL_READ_ONLY_OPTIMAL_KHR`
  - `VK_KHR_dynamic_rendering`
- in `VkRenderingInfo`, `pDepthAttachment` and `pStencilAttachment` are
  specified separately with some requirements
  - if both specify a image view, they must specify the same image view
  - `pDepthAttachment`'s image view must include a depth component
    - its layout must not be
      - `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
      - `VK_IMAGE_LAYOUT_STENCIL_ATTACHMENT_OPTIMAL`
      - `VK_IMAGE_LAYOUT_STENCIL_READ_ONLY_OPTIMAL`
      - IOW, must use depth
  - `pStencilAttachment`'s image view must include a stencil component
      - `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`
      - `VK_IMAGE_LAYOUT_DEPTH_ATTACHMENT_OPTIMAL`
      - `VK_IMAGE_LAYOUT_DEPTH_READ_ONLY_OPTIMAL`
      - IOW, must use stencil

## Point and Line Rasterization

- `VkPhysicalDeviceFeatures`
  - `largePoints` is true if point size greather than 1.0 is supported
  - `wideLines` is true if line width greather than 1.0 is supported
- `VkPhysicalDeviceLimits`
  - `pointSizeRange` and `pointSizeGranularity`
    - point size written to `PointSize` is clamped to the range
  - `lineWidthRange` and `lineWidthGranularity`
    - line width specified by `VkPipelineRasterizationStateCreateInfo` is
      clamped to the range
  - `strictLines`
    - line rasterization follows the strict rules or not
- point rasterization
  - same as a square centered at the point position, where the square width is
    the point size
- line rasterization
  - when `strictLines` is true, it is the same as a rectangle centered at the
    line, where two sides have length the same as line width and two sides
    have length the same as the line length
  - when `strictLines` is false, it is the same as a parallelogram centered at
    the line, with two exceptions
    - interpolations can follow either strict or non-strict rules
    - non-antialiased lines can follow the Bresenham rule instead
- `VK_EXT_line_rasterization`
  - all modes are optional, and support status is specified by
    `VkPhysicalDeviceLineRasterizationFeaturesEXT`
  - `VK_LINE_RASTERIZATION_MODE_RECTANGULAR_EXT` forces the strict rule
  - `VK_LINE_RASTERIZATION_MODE_BRESENHAM_EXT` forces the Bresenham rule
  - `VK_LINE_RASTERIZATION_MODE_RECTANGULAR_SMOOTH_EXT` is similar to the
    strict rule, except implementations can include pixels that are not
    covered and can compute coverage values freely
  - also support stippled line rasterization

## YCbCr Conversion

- feature bits
  - these control valid values of `xChromaOffset` and `yChromaOffset`
    - `VK_FORMAT_FEATURE_MIDPOINT_CHROMA_SAMPLES_BIT`
    - `VK_FORMAT_FEATURE_COSITED_CHROMA_SAMPLES_BIT`
    - each bit enables each chroma location
  - this controls valid values of `chromaFilter`
    - `VK_FORMAT_FEATURE_SAMPLED_IMAGE_YCBCR_CONVERSION_LINEAR_FILTER_BIT`
    - without the bit, only `VK_FILTER_NEAREST` is allowed
  - this controls valid values of `minFilter`, `magFilter`, and `chromaFilter`
    - `VK_FORMAT_FEATURE_SAMPLED_IMAGE_YCBCR_CONVERSION_SEPARATE_RECONSTRUCTION_FILTER_BIT`
    - without the bit, all three must have the same value
  - these controls valid values of `forceExplicitReconstruction`
    - `VK_FORMAT_FEATURE_SAMPLED_IMAGE_YCBCR_CONVERSION_CHROMA_RECONSTRUCTION_EXPLICIT_BIT`
      - with the bit, always explicit and `forceExplicitReconstruction` is
        assumed true
      - without the bit, it is implicit by default
    - `VK_FORMAT_FEATURE_SAMPLED_IMAGE_YCBCR_CONVERSION_CHROMA_RECONSTRUCTION_EXPLICIT_FORCEABLE_BIT`
      - with the bit, `forceExplicitReconstruction` can be set to true
- when CbCr is subsampled, reconstruction is required
  - reconstruction uses `chromaFilter`
    - `minFilter` and `magFilter` are not used
  - implicit reconstruction
    - scale unnormalized coordinates `(u, v)` to be `(u/2, v/2)`
    - if `VK_FILTER_NEAREST`, use the texel closest to `(u/2, v/2)`
    - if `VK_FILTER_LINEAR`, use the average of the 4 texels closest to `(u/2, v/2)`
  - explicit reconstruction
    - if `VK_FILTER_NEAREST` and texel `(i, j)` is needed, use texel `(i/2, j/2)`
    - if `VK_FILTER_LINEAR` and texel `(i, j)` is needed, use the average of
      the 4 texels closest to `(i/2, j/2)`
      - because `VK_FILTER_LINEAR` needs 4 texels, this can read up to 9
        texels in total
      - quad-linear filtering
  - chroma offsets (cosited or midpoint) are chosen based on how CbCr was
    subsampled
    - Y and full CbCr samples are co-sited
    - when full CbCr samples are downsampled,
      - we can store the average of each 2x2 samples, in this case we want to
        treat the stored sample as the midpoint of the 2x2 samples
      - we can store the top-left sample of each 2x2 samples, in this case we
        want to treat the stored sample as co-sited with the top-left Y aample

## Sparse Resources

- sparse binding
  - create the resource with `VK_IMAGE_CREATE_SPARSE_BINDING_BIT` or
    `VK_BUFFER_CREATE_SPARSE_BINDING_BIT`
  - use `vkQueueBindSparse` to bind memories
  - a resource is backed 1 or more memories
  - when a resource is active, it must be fully resident
  - when a resource is idle, memories can be unbound/rebound/moved
- sparse residency
  - create the resource with `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT` or
    `VK_BUFFER_CREATE_SPARSE_RESIDENCY_BIT`
  - when a resource is active, it can be partially resident
  - mip tail is the tail levels of a mipmap that cannot be partially resident

## `VK_ANDROID_external_memory_android_hardware_buffer`

- export ahb
  - `vkGetPhysicalDeviceImageFormatProperties2` with
    `VkPhysicalDeviceExternalImageFormatInfo` to query support
  - `vkCreateImage` with `VkExternalMemoryImageCreateInfo` to create the image
  - `vkAllocateMemory` with `VkExportMemoryAllocateInfo` to allocate the
    memory
  - `vkGetMemoryAndroidHardwareBufferANDROID` to export the AHB
  - in this path, only the public AHB formats are supported
- import ahb
  - `vkGetAndroidHardwareBufferPropertiesANDROID` with
    `VkAndroidHardwareBufferPropertiesANDROID` and
    `VkAndroidHardwareBufferFormatPropertiesANDROID` to query ahb
  - `vkCreateImage` with `VkExternalMemoryImageCreateInfo` and optionally
    `VkExternalFormatANDROID` to create the image
  - `vkAllocateMemory` with `VkImportAndroidHardwareBufferInfoANDROID` to
    import the ahb
  - in this path, ahb can have implemetation-defined formats
- allocate externally and import
  - `vkGetPhysicalDeviceImageFormatProperties2` with
    `VkAndroidHardwareBufferUsageANDROID` to get the optimal ahb usage
  - in this path, vk is consulted for the ahb usage

## Image Views

- a `VkImageView` can be created fro a `VkImage`
  - `VkImageViewType` and `VkImageType` can differ under conditions
    - `VK_IMAGE_VIEW_TYPE_xD` is always compatible with `VK_IMAGE_TYPE_xD` 
    - `VK_IMAGE_VIEW_TYPE_xD_ARRAY` is always compatible with `VK_IMAGE_TYPE_xD` 
    - `VK_IMAGE_VIEW_TYPE_CUBE` and `VK_IMAGE_VIEW_TYPE_CUBE_ARRAY` are
      compatible with `VK_IMAGE_TYPE_2D` if
      - `imageCubeArray` feature is supported and enabled
      - the image was created with `VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT`
      - view `layerCount` must be 6 or multiples of 6
    - `VK_IMAGE_VIEW_TYPE_2D` and `VK_IMAGE_VIEW_TYPE_2D_ARRAY` are compatible
      with `VK_IMAGE_TYPE_3D` if
      - `imageView2DOn3DImage` feature is supported and enabled
      - the image was created with `VK_IMAGE_CREATE_2D_ARRAY_COMPATIBLE_BIT`
    - there is also `VK_EXT_image_2d_view_of_3d`
  - view `format` and image `format` can differ if
    - the image was created with `VK_IMAGE_CREATE_MUTABLE_FORMAT_BIT`
  - view `subresourceRange` specifies a subset of the miptree accessible to
    the view
    - `baseMipLevel` and `levelCount` select the mip levels
    - `baseArrayLayer` and `layerCount` select the array layers
    - note that 3D images have only a single array layer
      - `VK_EXT_image_sliced_view_of_3d` allows selecting slices at a mip level
- VUIDs
  - VUID-VkImageViewCreateInfo-image-04970
    - `levelCount` must be 1 when creating a 2D view from a 3D image
  - VUID-VkImageViewSlicedCreateInfoEXT-None-07870
    - `levelCount` must be 1 when creating a sliced 3D view from a 3D image
- core
  - `imageView2DOn3DImage` feature indicates
    `VK_IMAGE_CREATE_2D_ARRAY_COMPATIBLE_BIT` support, permitting a 2D or 2D
    array image view of a 3D image
  - `VkFramebufferCreateInfo` does not allow 3D views.  The bit allows 3D
    images to be used as color buffers
  - it is designed for rendering, and is similar to
    `glFramebufferTextureLayer`
- `VK_EXT_image_2d_view_of_3d`
  - `image2DViewOf3D` feature indicates a `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE`
    descriptor can be a 2D image view of a 3D image
  - `sampler2DViewOf3D` feature indicates a `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE`
    descriptor can be a 2D image view of a 3D image
  - `VK_IMAGE_CREATE_2D_VIEW_COMPATIBLE_BIT_EXT` must be specified when
    creating the 3D image
  - the bit is for sampling, and is similar to `glBindImageTexture`
- `VK_EXT_image_sliced_view_of_3d`
  - `imageSlicedViewOf3D` indicates a `VK_DESCRIPTOR_TYPE_STORAGE_IMAGE`
    descriptor can be a sliced 3D image view of a 3D image

## `VK_EXT_attachment_feedback_loop_layout`

- feedback loop
  - when a subpass accesses a subresource as a color/depth/stencil attachment,
    and at the same time as an input attachment or an image resource, a
    feedback loop is formed
    - unless the subresource is not written to as a color/depth/stencil
      attachment
  - a subpass containing a feedback loop causes a data race, unless
    - a memory dependency between the read/write
    - `VK_EXT_rasterization_order_attachment_access`
- `VK_EXT_attachment_feedback_loop_layout`
  - this extension allows a feedback loop between a color/depth/stencil
    attachment and an image resource
    - that is, the extension replaces `VUID-vkCmdDraw-None-06538` by
      `VUID-vkCmdDraw-None-08753`
    - `VUID-vkCmdDraw-None-06538` states: If any recorded command in the
      current subpass writes to an image subresource as an attachment, this
      command must not read from the memory backing that image subresource in
      any other way than as an attachment
  - the device must have been created with `attachmentFeedbackLoopLayout`
  - the image must have been created with
    `VK_IMAGE_USAGE_ATTACHMENT_FEEDBACK_LOOP_BIT_EXT`
  - the pipeline must have been created with
    `VK_PIPELINE_CREATE_COLOR_ATTACHMENT_FEEDBACK_LOOP_BIT_EXT` or
    `VK_PIPELINE_CREATE_DEPTH_STENCIL_ATTACHMENT_FEEDBACK_LOOP_BIT_EXT`
  - the render pass must have been created with
    `VK_DEPENDENCY_FEEDBACK_LOOP_BIT_EXT`
  - the image must have been transitioned to
    `VK_IMAGE_LAYOUT_ATTACHMENT_FEEDBACK_LOOP_OPTIMAL_EXT`

## `VK_KHR_shader_float_controls`

- it has been promoted to 1.2
- `VkShaderFloatControlsIndependence`
  - `VK_SHADER_FLOAT_CONTROLS_INDEPENDENCE_32_BIT_ONLY`: float16/float64 share
    the same control
  - `VK_SHADER_FLOAT_CONTROLS_INDEPENDENCE_ALL`: float16, float32, and float64
    are independent
  - `VK_SHADER_FLOAT_CONTROLS_INDEPENDENCE_NONE`: float16/float32/float64
    share the same control
- `VkPhysicalDeviceFloatControlsProperties`
  - `denormBehaviorIndependence`
  - `roundingModeIndependence`
  - `shaderSignedZeroInfNanPreserveFloat{16,32,64}` indicates whether
    `SignedZeroInfNanPreserve` is supported
  - `shaderDenormPreserveFloat{16,32,64}` indicates whether `DenormPreserve`
    is supported
  - `shaderDenormFlushToZeroFloat{16,32,64}` indicates whether
    `DenormFlushToZero` is supported
  - `shaderRoundingModeRTEFloat{16,32,64}` indicates whether
    `RoundingModeRTE` is supported
  - `shaderRoundingModeRTZFloat{16,32,64}` indicates whether
    `RoundingModeRTZ` is supported

## Shader Data Type Widths

- `VkPhysicalDeviceFeatures` has
  - `shaderFloat64` specifies support for 64-bit floats
  - `shaderInt64` specifies support for 64-bit ints
  - `shaderInt16` specifies support for 16-bit ints
- `VkPhysicalDeviceVulkan11Features` has
  - `storageBuffer16BitAccess` specifies support for 16-bit load/store on
    ssbo, as well as width-only conversions of 16-bit types
  - `uniformAndStorageBuffer16BitAccess` is the same as above but on ubo
  - `storagePushConstant16` ditto
  - `storageInputOutput16` ditto
- `VkPhysicalDeviceVulkan12Features` has
  - `storageBuffer8BitAccess` specifies support for 8-bit load/store on ssbo,
    as well as width-only conversions of 8-bit types
  - `uniformAndStorageBuffer8BitAccess` is the same as above but on ubo
  - `storagePushConstant8` ditto
  - `shaderBufferInt64Atomics` specifies support for 64-bit int atomic
    operations on buffers
  - `shaderSharedInt64Atomics` specifies support for 64-bit int atomic
    operations on shared and payload memory
  - `shaderFloat16` specifies support for 16-bit floats
  - `shaderInt8` specifies support for 8-bit ints
