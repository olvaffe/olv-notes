Vulkan
======

## History

- 1.0 was released in 2016
  - HW features are about GL 4.3 or GLES 3.1
  - plus persistent and coherent memory
- 1.1 was released in 2018
- 1.2 was released in 2020
  - no new HW reqs
- 1.3 was released in 2022
  - no new HW reqs

## Chapter 1. Preamble

## Chapter 2. Introduction

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
- 3.9. Numeric Representation and Computation
  - only applies to computations outside of shader execution, , such as
    texture image specification and sampling, and per-fragment operations
    - requirements for shader execution differ and are specified by the
      Precision and Operation of SPIR-V Instructions section

## Chapter 4. Initialization

- 4.1. Command Function Pointers
  - `vkGetInstanceProcAddr` can query
    - global commands
      - `vkEnumerateInstanceVersion` since 1.1
      - `vkEnumerateInstanceExtensionProperties`
      - `vkEnumerateInstanceLayerProperties`
      - `vkCreateInstance`
      - instance must be `NULL`
    - self
      - instance can be NULL since 1.2
    - core dispatchable commands
    - enabled instance extension dispatchable commands
    - available device extension dispatchable commands
  - `vkGetDeviceProcAddr` can query
    - requested core version device-level dispatchable commands
      - core version is requested by `VkApplicationInfo::apiVersion`
      - with `maintenance5`, NULL must be returned for commands not in the
        requested core version
    - enabled extension device-level dispatchable commands
  - physical-device-level functionality
    - a functionality introduced since a core version can be used if
      `VkPhysicalDeviceProperties::apiVersion` is the same or newer than the
      core version
    - a functionality introduced with an instance extension can be used if the
      instance extension is enabled
    - a functionality introduced with a device extension can be used if the
      device extension is available
- 4.2. Instances
  - `VkApplicationInfo::apiVersion` must be the highest version of Vulkan that
    the application is designed to use
    - The patch version number specified in `apiVersion` is ignored
    - The variant version of the instance must match that requested in `apiVersion`
    - Because Vulkan 1.0 implementations may fail with
      `VK_ERROR_INCOMPATIBLE_DRIVER` when `apiVersion` is not 1.0,
      applications should determine the version of Vulkan available before
      calling `vkCreateInstance`. If the `vkGetInstanceProcAddr` returns NULL
      for `vkEnumerateInstanceVersion`, it is a Vulkan 1.0 implementation.
      Otherwise, the application can call `vkEnumerateInstanceVersion` to
      determine the version of Vulkan.
    - As long as the instance supports at least Vulkan 1.1, an application can
      use different versions of Vulkan with an instance than it does with a
      device or physical device.
      - that is, the instance-level functionality is capped to
        `min(VkApplicationInfo::apiVersion, vkEnumerateInstanceVersion)`
      - the physical-device-level and device-level functionality is capped to
        `min(VkApplicationInfo::apiVersion, VkPhysicalDeviceProperties::apiVersion)`
    - in practice, roughly
      - the loader decides the instance version
      - the driver decides the device version
      - when the loader supports instance version 1.0, it also limits the
        device version to 1.0
      - otherwise, the two versions are orthogonal
  - `VK_EXT_validation_features`
    - deprecates `VK_EXT_validation_flags`
    - enable/disable specific validation features of
      `VK_LAYER_KHRONOS_validation`
      - thread safety
      - api params
      - object lifetimes
      - core checks
      - unique handles

## Chapter 5. Devices and Queues

- 5.1. Physical Devices
  - `VkPhysicalDevice` is used to query device caps and features
- 5.2. Devices
  - `VkDevice` and the available `VkQueue`'s are created together

## Chapter 6. Command Buffers

- secondary command buffer inherits no state from the primary command
  buffer
  - except for render pass and subpass
  - secondary command buffer must explicitly set all other states
- 6.2. Command Pools
  - A command pool is almost a dummy object to reset/free a group of command
    buffers in one go
  - A command buffer consists of
    - a list of BOs, dynamically growing, with one chained to another
    - a vector of reloc entries (if resources may move on the impl)
    - a vector of binding table blocks
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
  - `VkSemaphore` is for cross-queue synchronization with submit granularity
  - `VkFence` is for vk/cpu synchronization with submit granularity
    - wait is on the CPU side
  - `VkEvent` is for vk/cpu synchronization with command granularity
    - wait is on the GPU side
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
  - Within a single queue, vkQueueSubmit are executed in call order
  - Within a VkQueueSubmit, VkSubmitInfo are executed in array order
  - Within a VkSubmitInfo, VkCommandBuffer are executed in array order
  - Within a VkCommandBuffer, commands outside of render passes are executed in
    recorded order (this includes vkCmdBeginRenderPass and vkCmdEndRenderPass)
  - Within a render pass subpass, commands are executed in recorded order
  - Between render pass subpasses, there is no implicit ordering
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

- 8.1. Render Pass Objects
  - `VkRenderPass` describes a rendering pass abstractly; no image but only their
    formats, etc.
  - `VkFramebuffer` is created from VkRenderPass and a list of VkImageView as the
    attachments
    - unless `VK_FRAMEBUFFER_CREATE_IMAGELESS_BIT`
    - `VkFramebuffer` is like a physical instance of a `VkRenderPass`
    - the separation is such that, when a pipeline is created for a render pass,
      the pipeline can work with any framebuffer compatible with the render pass.
  - A `VkRenderPass` contains a set of (abstract) attachments it will work on
    - each attachment is described with `VkAttachmentDescription` abstractly
    - format
    - load/store ops
    - initial layout: the layout the attachment is in when entering the render
      pass
    - final layout: the layout the attachment will be transitioned to by the
      implementation automatically when exiting the render pass
  - A subpass of a render pass works with a subset of the attachments
    - input attachemnts
    - color attachemnts
    - resolve attachemnts
    - depth attachemnt
    - preserve attachemnts
    - the implementation transitions all attachments to the specified layouts
      automatically when entering a subpass
- 8.3. Render Pass Compatibility
  - framebuffers and graphics pipelines are created based on a specific render
    pass object
  - during rendering, framebuffers and graphics pipelines can be used to any
    compatible render pass object
    - two attachment references are compatible if they have the same format
      and sample count (or are both `VK_ATTACHMENT_UNUSED`)
    - two arrays of attachment references are compatible if all pairs of
      attachment references are compatible (when the array sizes mismatch,
      missing attachment references are considered `VK_ATTACHMENT_UNUSED`)
    - two render passes are compatible if all attachment references (color,
      ds, input, resolve) are compatible, and they are otherwise idential
      except for
      - initial and final image layouts of attachments
      - load and store ops of attachments
      - image layouts in attachment references
    - as a special case, if two render passes have a single subpass,
      - the ds and resolve attachment references are ignored for compatibility
        check
- 8.7. Render Pass Multisample Resolve Operations
  - whether the attachments are color or depth/stencil,
    - the pipelne stage is `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT`
    - the read access is `VK_ACCESS_COLOR_ATTACHMENT_READ_BIT`
    - the write access is `VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT`
- 8.8. Render Pass Commands
  - depending on VkSubpassContents, any subpass of a render pass either
    execute only commands in the primary command buffer or only command the
    secondary command buffers
- 8.10. Common Render Pass Data Races (Informative)
  - this section summarizes renderpass feedback loops
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

## Chapter 9. Shaders

- 9.24. Static Use
  - `OpVariable` declares a variable which is a pointer to a global object in
    memory
  - if an entry point's call tree contains a funcition that contains an
    instruction using the pointer, the entry point statically uses the object
  - a shader entry point also statically uses all variables explicitly
    declared in its interface

## Chapter 10. Pipelines

- A VkPipelineLayout consists of multiple descriptor set layouts and push
  constants.
  - Because a pipeline can work with multiple sets at the same time.
- A pipeline is created from a pipeline layout and many other states
  - renderPass / subpass
  - pipeline cache
  - base pipeline
- `VK_KHR_pipeline_library`
  - a pipeline library is a `VkPipeline` that cannot be used directly
  - to create a pipeline library, specify `VK_PIPELINE_CREATE_LIBRARY_BIT_KHR`
    when creating the pipeline
  - to create a pipeline from pipeline libraries,
    `VkPipelineLibraryCreateInfoKHR` to `VkGraphicsPipelineCreateInfo`
- `VK_EXT_graphics_pipeline_library`
  - init
    - `VkGraphicsPipelineLibraryCreateInfoEXT`
      - `graphicsPipelineLibrary` is true if gpl is supported
      - this feature query exists because the extension might be promoted to core
    - `VkPhysicalDeviceGraphicsPipelineLibraryPropertiesEXT`
      - `graphicsPipelineLibraryFastLinking` is true if creating a pipeline from
        pipeline libraries is cheap without
        `VK_PIPELINE_CREATE_LINK_TIME_OPTIMIZATION_BIT_EXT`
      - `graphicsPipelineLibraryIndependentInterpolationDecoration` is true if
        interpolation decorations of vs outs and fs ins do not need to match
  - shader module is deprecated
    - `VkShaderModuleCreateInfo` can be chained to `VkPipelineShaderStageCreateInfo` directly
  - pipeline layout
    - `VkPipelineLayoutCreateInfo` gains a new flag,
      `VK_PIPELINE_LAYOUT_CREATE_INDEPENDENT_SETS_BIT_EXT`
    - vs and fs may need different descriptor sets.  When compiling them
      independently, `VkPipelineLayout` must be specified twice
      - one option is to use a full `VkPipelineLayout` and specify the same
        layout
      - another option is to use two partial `VkPipelineLayout`s, which must be
        created with `VK_PIPELINE_LAYOUT_CREATE_INDEPENDENT_SETS_BIT_EXT`
  - pipeline
    - `VkGraphicsPipelineLibraryCreateInfoEXT`
      - `flags` specifies a subset of the graphics pipeline to compile
        - there are 4 parts
        - `VK_GRAPHICS_PIPELINE_LIBRARY_VERTEX_INPUT_INTERFACE_BIT_EXT`
        - `VK_GRAPHICS_PIPELINE_LIBRARY_PRE_RASTERIZATION_SHADERS_BIT_EXT`
        - `VK_GRAPHICS_PIPELINE_LIBRARY_FRAGMENT_SHADER_BIT_EXT`
        - `VK_GRAPHICS_PIPELINE_LIBRARY_FRAGMENT_OUTPUT_INTERFACE_BIT_EXT`
      - when this struct is omitted,
        - if not doing pipeline libraries, `flags` is assumed to include all bits
        - if doing pipeline libraries, `flags` is assumed to include no bit
    - `VkGraphicsPipelineCreateInfo` gains 2 new flags
      - `VK_PIPELINE_CREATE_RETAIN_LINK_TIME_OPTIMIZATION_INFO_BIT_EXT`
        specifies that the pipeline library being created should retain info for
        LTO later
      - `VK_PIPELINE_CREATE_LINK_TIME_OPTIMIZATION_BIT_EXT` specifies that the
        pipeline being created should have LTO applied on the pipeline libraries
        - on the other hand, when the flag is omitted, the pipeline should be
          fast-linked
- 10.1. Compute Pipelines
  - `VkComputePipelineCreateInfo` mainly consists of
    - `VkPipelineCreateFlags`
      - `VK_PIPELINE_CREATE_DISABLE_OPTIMIZATION_BIT` disables shader
        optimization
      - `VK_PIPELINE_CREATE_ALLOW_DERIVATIVES_BIT` allows other pipelines to
        derive from this one (to reduce compile time)
      - `VK_PIPELINE_CREATE_DERIVATIVE_BIT` allows this pipeline to derive
        from another one (to reduce compile time)
      - `VK_PIPELINE_CREATE_VIEW_INDEX_FROM_DEVICE_INDEX_BIT`
      - `VK_PIPELINE_CREATE_DISPATCH_BASE_BIT` allows dispatching with
        `vkCmdDispatchBase`
      - `VK_PIPELINE_CREATE_FAIL_ON_PIPELINE_COMPILE_REQUIRED_BIT` early
        returns with `VK_PIPELINE_COMPILE_REQUIRED` if compilation is required
        (as opposed to pipeline cache hit)
      - `VK_PIPELINE_CREATE_EARLY_RETURN_ON_FAILURE_BIT` sets the rest of the
        pipelines to `VK_NULL_HANDLE` on the first failure of multiple
        pipeline creation
      - there are more bits that are not in core
    - `VkPipelineShaderStageCreateInfo`
    - `VkPipelineLayout` describes the pipeline layout (descriptor set
      layouts, push constants, etc.)
      - the layout is "consumed", in the sense that the pipeline can outlive
        the layout
      - impl needs the layout because
        - shader refers to the descriptor with `set` and `binding`
        - impl uses the layout to, for example, map `set` and `binding` to the
          offset of the descriptor
- 10.2. Graphics Pipelines
  - `VkGraphicsPipelineCreateInfo` is divided into 4 state groups
    - Vertex Input State
      - this state group is required for a complete graphipcs pipeline if the
        pipeline has vertex shader
      - `VkPipelineVertexInputStateCreateInfo`
      - `VkPipelineInputAssemblyStateCreateInfo`
    - Pre-Rasterization Shader State
      - this state group is always required for a complete graphics pipeline
      - `VkPipelineShaderStageCreateInfo` for vs/tcs/tes/gs/task/mesh
      - `VkPipelineLayout`
      - `VkPipelineViewportStateCreateInfo`
      - `VkPipelineRasterizationStateCreateInfo`
      - `VkPipelineTessellationStateCreateInfo`
      - `VkRenderPass` and `subpass`
        - or `viewMask` of `VkPipelineRenderingCreateInfo`
      - some extensions
    - Fragment Shader State
      - this state group is required for a complete graphipcs pipeline if
        `rasterizerDiscardEnable` is true
      - `VkPipelineShaderStageCreateInfo` for fs
      - `VkPipelineLayout`
      - `VkPipelineMultisampleStateCreateInfo`
        - if sample shading is enabled or `renderpass` is not `VK_NULL_HANDLE`
      - `VkPipelineDepthStencilStateCreateInfo`
      - `VkRenderPass` and `subpass`
        - or `viewMask` of `VkPipelineRenderingCreateInfo`
      - some extensions and some flags
    - Fragment Output State
      - this state group is required for a complete graphipcs pipeline if
        `rasterizerDiscardEnable` is true
      - `VkPipelineColorBlendStateCreateInfo`
      - `VkRenderPass` and `subpass`
        - or `VkPipelineRenderingCreateInfo`
      - `VkPipelineMultisampleStateCreateInfo`
      - some extensions and some flags
  - `renderPass` and `subpass`
    - unless dynamic rendering, `renderPass` and `subpass` must be valid
    - if dynamic rendering, `VkPipelineRenderingCreateInfo` is provided in
      place of `renderPass`
      - if both are specified, `VkPipelineRenderingCreateInfo` is ignored
- 10.3. Ray Tracing Pipelines
- 10.5. Multiple Pipeline Creation
  - the create functions can create multiple pipelines at a time
  - if it fails to create for a pipeline, the object will be set to
    `VK_NULL_HANDLE` and the create function will return the error
    - if multiple pipelines fails, it returns the error code of any of the
      failed creation
- 10.7. Pipeline Cache
  - `VkPipelineCache` is for faster pipeline creation

## Chapter 11. Memory Allocation

- 11.2. Device Memory
  - `VK_ANDROID_external_memory_android_hardware_buffer`
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

## Chapter 12. Resource Creation

- 12.1. Buffers
  - `VkBufferCreateInfo`
    - `size` must be positive and must not exceed
      `VkPhysicalDeviceMaintenance4Properties::maxBufferSize`
    - `sharingMode` is explained in 12.9
    - `flags`
      - `VK_BUFFER_CREATE_SPARSE_BINDING_BIT` depends on `sparseBinding`
        feature
      - `VK_BUFFER_CREATE_SPARSE_RESIDENCY_BIT` depends on
        `sparseResidencyBuffer` feature
      - `VK_BUFFER_CREATE_SPARSE_ALIASED_BIT` depends on
        `sparseResidencyAliased` feature
      - `VK_BUFFER_CREATE_PROTECTED_BIT` depends on `protectedMemory` feature
      - sparse and protected are mutually exclusive
    - `usage`
      - all valid usage is supported (because buffers have no format)
      - they determine where the buffers can be used
        - e.g., `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` allows the buffer to be
          used in `vkCmdBindVertexBuffers`
- 12.2. Buffer Views
  - `VkBufferViewCreateInfo`
    - `buffer`
      - must have been created with `VK_BUFFER_USAGE_UNIFORM_TEXEL_BUFFER_BIT`
        or `VK_BUFFER_USAGE_STORAGE_TEXEL_BUFFER_BIT`
      - if not sparse, must have a memory bound
    - `offset`
      - must be less than the buffer size
      - must be aligned to
        `VkPhysicalDeviceLimits::minTexelBufferOffsetAlignment`
    - `range`
      - must be positive and less than the buffer size, or `VK_WHOLE_SIZE`
      - element count must not exceed
        `VkPhysicalDeviceLimits::maxTexelBufferElements`
    - `VkBufferUsageFlags2CreateInfoKHR::usage`
      - if exists, must be a subset of the usage of the buffer
      - otherwise, the usage of the buffer is assumed
      - `VK_BUFFER_USAGE_UNIFORM_TEXEL_BUFFER_BIT` depends on
        `VK_FORMAT_FEATURE_UNIFORM_TEXEL_BUFFER_BIT`
      - `VK_BUFFER_USAGE_STORAGE_TEXEL_BUFFER_BIT` depends on
        `VK_FORMAT_FEATURE_STORAGE_TEXEL_BUFFER_BIT`
- 12.3. Images
  - `VK_IMAGE_TILING_LINEAR` may (or may not) have more restrictions
    - as reported by `vkGetPhysicalDeviceFormatProperties` or
      `vkGetPhysicalDeviceImageFormatProperties2`?
    - `imageType` must be `VK_IMAGE_TYPE_2D`
    - `format` must not be a depth/stencil format
    - `mipLevels` must be 1
    - `arrayLayers` must be 1
    - `samples` is `VK_SAMPLE_COUNT_1_BIT`
    - `usage` must only include `VK_IMAGE_USAGE_TRANSFER_SRC_BIT` and/or
      `VK_IMAGE_USAGE_TRANSFER_DST_BIT`
    - other implementation-defined restrictions
  - formats that reuqire ycbcr conversion may (or may not) have similar
    restrictions
  - Image Creation Limits
    - `imageCreateMaybeLinear` is derived from tiling
    - `imageCreateFormatFeatures` is derived from tiling and
      `vkGetPhysicalDeviceFormatProperties`
    - `imageCreateImageFormatPropertiesList` is derived from from calling
      `vkGetPhysicalDeviceImageFormatProperties2`
      - it is a list and there can be more than one entry when the image has
        multiple external handle types, etc.
      - it is empty if any call fails
    - `imageCreateMaxMipLevels`, `imageCreateMaxArrayLayers`,
      `imageCreateMaxExtent`, and `imageCreateSampleCounts` are derived from
      `imageCreateImageFormatPropertiesList`
    - as the most basic rule, for an image creation to be valid,
      `imageCreateImageFormatPropertiesList` must be non-empty
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
  - DRM modifiers
    - `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT` mixes tiling and layout
      together.  Format data and format metadata reside on different memory
      planes
    - a single-plane format thus can have multiple memory planes when the tiling is
      `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT`
    - can we combine multi-planar format with drm modifier?  It has `N*M`
      planes in theory.
  - `VK_EXT_image_compression_control`
    - `VK_EXT_image_compression_control` adds `imageCompressionControl` feature
    - `VkImageCompressionPropertiesEXT`
      - it can be chained to
        - `vkGetPhysicalDeviceImageFormatProperties2`
        - `vkGetImageSubresourceLayout2KHR`
      - `imageCompressionFlags` is one of
        - `VK_IMAGE_COMPRESSION_DEFAULT_EXT` to indicate lossless compression
        - `VK_IMAGE_COMPRESSION_FIXED_RATE_EXPLICIT_EXT` to indicate lossy
          compression
        - `VK_IMAGE_COMPRESSION_DISABLED_EXT` to indicate no compression
      - `imageCompressionFixedRateFlags` is 0 or one of
        - `VK_IMAGE_COMPRESSION_FIXED_RATE_1BPC_BIT_EXT` to
          `VK_IMAGE_COMPRESSION_FIXED_RATE_24BPC_BIT_EXT`
        - bpc stands for bit-per-component
    - `VkImageCompressionControlEXT`
      - it can be chained to
        - `VkImageCreateInfo`
        - `VkSwapchainCreateInfoKHR`
        - `VkPhysicalDeviceImageFormatInfo2`
      - `flags`
        - `VK_IMAGE_COMPRESSION_DEFAULT_EXT` is the default and allows
          lossless/no compression
        - `VK_IMAGE_COMPRESSION_FIXED_RATE_DEFAULT_EXT` and
          `VK_IMAGE_COMPRESSION_FIXED_RATE_EXPLICIT_EXT` allows
          lossless/lossy/no compression
        - `VK_IMAGE_COMPRESSION_DISABLED_EXT` allows no compression
      - `compressionControlPlaneCount` must match the plane count of the format
      - `pFixedRateFlags` is an array of masks that indicate the requested bpc of
        each plane
  - `VK_EXT_image_compression_control_swapchain`
    - it adds `imageCompressionControlSwapchain` feature
    - `vkGetPhysicalDeviceSurfaceFormats2KHR` can query
      `VkImageCompressionPropertiesEXT` to know if the wsi supports
      lossless/lossy/none compression for a surface format
    - `vkCreateSwapchainKHR` can include `VkImageCompressionControlEXT` to
      control compression for the swapchain images
- 12.4. Image Layouts
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
- 12.5. Image Views
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
    - if the image format is multi-planr (including sub-sampled), and the
      aspect is `VK_IMAGE_ASPECT_COLOR_BIT`, and the usage includes
      `VK_IMAGE_USAGE_SAMPLED_BIT`,
      - both the sampler and this image view must be created with
        `VkSamplerYcbcrConversionInfo`
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
  - `VK_IMAGE_CREATE_BLOCK_TEXEL_VIEW_COMPATIBLE_BIT`
    - it was added by `VK_KHR_maintenance2`
    - there has always been `VUID-VkImageViewCreateInfo-image-07072` (which was
      named `VUID-VkImageViewCreateInfo-image-01584` at first) which states
      - If image was created with the
        `VK_IMAGE_CREATE_BLOCK_TEXEL_VIEW_COMPATIBLE_BIT` flag and format is a
        non-compressed format, the `levelCount` and `layerCount` members of
        `subresourceRange` must both be 1
    - in 1.2.171, `VUID-VkImageViewCreateInfo-image-04739` was added to ban
      `VK_IMAGE_VIEW_TYPE_3D`
    - in 1.3.226, `VUID-VkImageViewCreateInfo-image-04739` was removed to allow
      `VK_IMAGE_VIEW_TYPE_3D`
  - `VkImageAspectFlagBits`
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
- 12.8. Resource Memory Association
  - `VK_IMAGE_CREATE_DISJOINT_BIT`
    - driver by default allocates a buffer and puts planes on different region of
      the same buffer
      - thus only a `VkDeviceMemory` needs to be bound
    - when `VK_IMAGE_CREATE_DISJOINT_BIT` is specified, driver puts different
      planes on different buffers
      - thus multiple `VkDeviceMemory` need to be bound
    - as such, internally, a `VkImage` can point to multiple `VkDeviceMemory`
      which respectively point to different BOs
- 12.9. Resource Sharing Mode
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

## Chapter 13. Samplers

- `VkSampler` describes a sampler (min filter, etc)
- 13.1. Sampler Y′CBCR Conversion
  - Conversion must be fixed at pipeline creation time, through use of a
    combined image sampler with an immutable sampler in
    `VkDescriptorSetLayoutBinding`
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

## Chapter 14. Resource Descriptors

- 14.1. Descriptor Types
  - samplers
  - sampled images
  - storage images
  - uniform texel buffers
  - storage texel buffers
  - uniform buffers
  - storage buffers
  - input attachments (color attachments being read in following subpasses)
- 14.2. Descriptor Sets
  - `VkDescriptorSetLayout` describes the layout (bindings) of a descriptor set
  - `VkPipelineLayout` is a group of descriptor set layouts
  - A `VkDescriptorPool` allocates enough memory (host or device, depending on
    the implementations) to hold the specified amount of descriptors of the
    specified types.
  - A `VkDescriptorSet` grabs some of the descriptors from the pool according to
    its `VkDescriptorLayout`.
  - A `VkDescriptorSetLayout` of a set has multiple bindings, with each binding
    corresponding to an array of descriptors of the same type.
  - `vkUpdateDescriptorSets` writes the descriptors into the memory
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

## Chapter 15. Shader Interfaces

- 15.8. Shader Resource Interface
  - 15.8.3 has a big note on variables sharing the same `DescriptorSet` and
    `Binding`

## Chapter 16. Image Operations

## Chapter 17. Fragment Density Map Operations

## Chapter 18. Queries

- 18.1. Query Pools
  - a `VkQueryPool` can hold `queryCount` queries of type `queryType`, both
    specified by `VkQueryPoolCreateInfo`
- 18.2. Query Operation
  - each query in a query pool has
    - a "status" that is either available or unavailable
    - a "state" that holds numerical values
    - both "status" and "state" are undefined initially
  - `vkCmdResetQueryPool` or `vkResetQueryPool` resets queries in a query pool
    - a reset sets "status" to unavailable and makes "state" undefined
  - `vkCmdBeginQuery` and `vkCmdEndQuery`
    - a query must begin and end in the same command buffer
      - conceptually, because there are secondary command buffers and
        `inheritedQueries` feature
    - a query must either
      - begin and end in the same subpass or
      - begin and end outside of a render pass
    - when `multiview` is enabled, a query must begin and end in the same
      subpass
      - the commands begin/end N queries at the same time, where N is the
        number of views
    - after begin, a query is considered active
      - that is, after the begin command is executed on the queue, not after
        the call returns
      - an active query must not be reset
      - two queries of the same type cannot be active at the same time
        - this rule says that the hw can support 1 outstanding active query
          for each type at a time
    - after end, the "status" of the query becomes available
      - only an active query can be ended
  - `vkCmdCopyQueryPoolResults` or `vkGetQueryPoolResults` retrieves and
    formats the results
    - a query in a pool is just some storage for some opaque data
    - these commands parse the opaque data, convert them to a standard
      format, and write the results to the specified destination
    - `VK_QUERY_RESULT_64_BIT` controls the results are in 64-bit or 32-bit uints
    - `VK_QUERY_RESULT_WAIT_BIT` waits for `vkCmdEndQuery` of all specified
      queries to be executed on the queue
      - that is, it waits until all specified queries become available
      - without it, and if any of the queries is unavailable, see
        `VK_QUERY_RESULT_PARTIAL_BIT`
    - `VK_QUERY_RESULT_WITH_AVAILABILITY_BIT` controls whether an additional
      64-bit or 32-bit uint is written per query to indicate availability
      (non-zero is available and 0 is unavailable)
    - `VK_QUERY_RESULT_PARTIAL_BIT` controls handling of unavailable queries
      - if a query is available, the result is final and well-defined
      - if a query is unavailable, and if `VK_QUERY_RESULT_PARTIAL_BIT` is
        set, the result is between 0 and the final value
      - otherwise, the call may
        - fail and return `VK_NOT_READY`
        - succeed and the result is undefined
- 18.3. Occlusion Queries
  - `VK_QUERY_TYPE_OCCLUSION` counts samples that pass the per-fragment tests
  - the result is a 32-bit or 64-bit uint
  - if `VK_QUERY_CONTROL_PRECISE_BIT` is not set, the result can be any
    non-zero value if any sample passes the per-fragment tests
- 18.4. Pipeline Statistics Queries
  - it queries a specified set of `VkPipeline` counters
    - IA, VS, TCS/TES, GS, CL, CS, etc.
    - see `VkQueryPipelineStatisticFlagBits`
  - some counters might be inaccurate
    - Vertices corresponding to incomplete primitives may contribute to the
      count.
    - Incomplete primitives may be counted.
    - The actual number of primitives output by the primitive clipping stage
      for a particular input primitive is implementation-dependent...'
    - Implementations may skip the execution of certain compute shader
      invocations or execute additional compute shader invocations for
      implementation-dependent reasons as long as the results of rendering
      otherwise remain unchanged.
- 18.5. Timestamp Queries
  - `VK_QUERY_TYPE_TIMESTAMP` reads the gpu clock counter
  - the result is a 32-bit or 64-bit uint
  - the counter may reset anytime except intra-command buffer or when
    `VK_EXT_calibrated_timestamps` is enabled
- 18.6. Performance Queries
  - `VK_KHR_performance_query` mainly adds
    `VK_QUERY_TYPE_PERFORMANCE_QUERY_KHR` query type
    - the query type usually reads counter values at begin and end
  - features and properties
    - `performanceCounterQueryPools` supports perf queries
    - `performanceCounterMultipleQueryPools` supports multiple query pools for
      perf queries
    - `allowCommandBufferQueryCopies` supports `vkCmdCopyQueryPoolResults` for
      perf queries
  - `vkAcquireProfilingLockKHR` and `vkReleaseProfilingLockKHR` 
    - to avoid counter value reset due to device suspend, the profiling lock
      must be held
    - strangely, it must be held before `vkBeginCommandBuffer`
  - `vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR`
    - enumerates available counters on a queue family
    - `VkPerformanceCounterKHR`
      - `unit` is generic, percentage, ns, bytes, bytes-per-second, kelvin,
        watts, volts, amps, hertz, cycles, etc.
      - `scope` is cmdbuf, renderpass, command, etc.
      - `storage` is int32/64, uint32/64, float32/64
      - `uuid` is the uuid of the counter
    - `VkPerformanceCounterDescriptionKHR`
      - `flags` is for various things such as if a counter affects perf
      - `name`/`category`/`description`
  - `vkGetPhysicalDeviceQueueFamilyPerformanceQueryPassesKHR`
  - `vkCreateQueryPool`
    - `VkQueryPoolPerformanceCreateInfoKHR` specifies the counters to enable
  - `vkQueueSubmit`
    - `VkPerformanceQuerySubmitInfoKHR` specifies the counter pass index
- 18.7. Transform Feedback Queries
  - `VK_QUERY_TYPE_TRANSFORM_FEEDBACK_STREAM_EXT` counts primitives written
    out by the SO hw as well as primitives that reach the SO hw
  - the result is two 32-bit or 64-bit uints
    - first one is count of prims written
    - second one is count of prims that would be written if the buffer was
      large enough
- 18.8. Primitives Generated Queries
  - `VK_QUERY_TYPE_PRIMITIVES_GENERATED_EXT` counts primitives generated,
    regardless of whether they reach the SO hw or not

## Chapter 19. Clear Commands

## Chapter 20. Copy Commands

## Chapter 21. Drawing Commands

## Chapter 22. Fixed-Function Vertex Processing

## Chapter 23. Tessellation

## Chapter 24. Geometry Shading

## Chapter 25. Mesh Shading

## Chapter 26. Cluster Culling Shading

## Chapter 27. Fixed-Function Vertex Post-Processing

- 27.4. Primitive Clipping
  - view volume
    - it is `[-w, w]` for x and y
    - it is `[0, w]` for z by default
  - cull volume and user-defined culling
    - `gl_CullDistance` is an array of `maxCullDistances` floats
      - positive is inside, zero is on, and negative is outside
    - if `gl_CullDinstance[i]` is negative for all vertices of a primitive, the
      primitive is discarded
  - clip volume and user-defined clipping
    - `gl_ClipDistance` is an array of `maxClipDistances` floats
      - positive is inside, zero is on, and negative is outside
    - if `gl_ClipDinstance[i]` is negative for any vertex of a primitive, the
      primitive is clipped
- 27.7. Coordinate Transformations
  - clip coordinates
    - this is the coordinates of `gl_Position`
  - normalized device coordinates
    - in clip coordinates, dividing xyz by w gives NDC
  - framebuffer coordinates
    - a viewport is defined by
      - `x` and `y`
      - `width` and `height`
      - `minDepth` and `maxDepth`
    - ndc coordinates are mapped to framebuffer coordinates using viewports
- knobs for z
  - `VK_EXT_depth_clip_control` has `negativeOneToOne`
    - when true, the view volume for z is `[-w, w]`
  - core has `depthClamp`
    - when false, z is clipped by the clip volume
    - when true, z is not clipped and is instead clamped by viewports
  - `VK_EXT_depth_clip_enable` has `depthClipEnable`
    - this provides explicit control for clipping, taking precedence over
      `depthClamp`
  - `VK_EXT_depth_range_unrestricted`
    - viewport `minDepth` and `maxDepth` must be in `[0.0, 1.0]` by default
    - when this extension is supported, they can be arbitrary
    - this does not affect depth clipping, but only depth clamping

## Chapter 28. Rasterization

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
- 28.10. Points
  - point rasterization
    - same as a square centered at the point position, where the square width is
      the point size
- 28.11. Line Segments
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
- 28.12. Polygons
  - depth bias
    - an offset (bias) can be added to z of framebuffer coordinates
  - knobs for depth bias
    - core has `depthBiasEnable`
      - when false, no bias is applied
    - core has `depthBiasConstantFactor`, `depthBiasClamp`, `depthBiasSlopeFactor`
      - the bias is roughly given by
        `m * depthBiasSlopeFactor + r * depthBiasConstantFactor`, clampped by
        `depthBiasClamp`
      - `m` is the slope of z
      - `r` is implementation-defined
    - `VK_EXT_depth_bias_control` has `depthBiasRepresentation` and
      `depthBiasExact`
      - they give explicit control over `r`

## Chapter 29. Fragment Operations

- coverage mask
  - there is a coverage mask associated with each fragment produced by
    rasterization
  - if a frag op results in a coverage mask of 0, the fragment is discard and
    no further frag ops are performed
- 29.1. Discard Rectangles Test
  - `VK_EXT_discard_rectangles`
  - if a sample falls inside or outside (depending on
    `VkDiscardRectangleModeEXT`) any discard rectangle, it is discarded
- 29.2. Scissor Test
  - if a sample falls outside the scissor rectangle, it is discarded
- 29.3. Exclusive Scissor Test
  - `VK_NV_scissor_exclusive`
  - if a sample falls inside the scissor rectangle, it is discarded
- 29.4. Sample Mask Test
  - samples not in `VkPipelineMultisampleStateCreateInfo::pSampleMask` are
    discarded
- 29.5. Fragment Shading
  - fragment shaders are invoked for each fragment, subjected to these rules
    - a fs must not be executed if the fragment has been discarded
    - a fs may not be executed if it has no side-effect
      - e.g., the fragment will be discarded and the fs has no storage op
      - or, a following fragment will override all side-effects
    - otherwise, a fs must be executed and
      - if sampling shading is enabled and requires multiple executions, the
        fs must be executed as specified
      - Each covered sample must be included in at least one fragment shader invocation.
  - `SampleMask`
    - reading returns samples covered by this execution
    - writing updates the sample mask (similar to
      `VkPipelineMultisampleStateCreateInfo::pSampleMask`)
  - `VK_EXT_shader_tile_image` is used with dynamic rendering to replace
    render pass
  - depth replacement
    - it replaces z in framebuffer coordinates by `gl_FragDepth`
  - `FragDepth`
    - writing replaces the depth value for all samples in `SampleMask`
  - `VK_EXT_shader_stencil_export`
  - `VK_EXT_fragment_shader_interlock`
- 29.6. Multisample Coverage
  - samples not in `SampleMask` are discarded
  - `alphaToCoverageEnable` and `alphaToOneEnable`
    - only applies to output 0 and only applies to float
    - `alphaToCoverageEnable` generates a sample mask based on the alpha value
      of the fragment, and is ANDed with the coverage mask
    - `alphaToOneEnable` replaces alpha value by 1
- 29.7. Depth and Stencil Operations
- 29.8. Depth Bounds Test
  - samples whose corresponding depth values in the attachment are outside of
    `[minDepthBounds, maxDepthBounds]` are discarded
  - when enabled, and if z value in the depth attachment is outside of
    `[minDepthBounds, maxDepthBounds]`, the sample is discarded
- 29.9. Stencil Test
- 29.10. Depth Test
  - depth clamping and range adjustment
    - z in framebuffer coordinates is a float
      - with depth clipping or depth clamping, and with sane `gl_FragDepth`,
        it should be between `[minDepth, maxDepth]`
      - otherwise, it is arbitrary
    - `VK_EXT_depth_clamp_zero_one`
      - when enabled,
        - if the attachment is D32 float, z is unchanged
        - otherwise, z is clamped to `[0.0, 1.0]`
      - when disabled,
        - if z is outside of `[minDepth, maxDepth]`, it becomes undefined
        - if the attachment is unorm and z is outside of `[0.0, 1.0]`, it
          becomes undefined
  - depth comparison
    - core has `depthTestEnable` and `depthCompareOp`
  - depth attachment write
    - core has `depthWriteEnable`
- 29.11. Representative Fragment Test
  - `VK_NV_representative_fragment_test`
- 29.12. Sample Counting
  - occulusion query counter is incremented for each sample
- 29.13. Fragment Coverage To Color
  - `VK_NV_fragment_coverage_to_color`
- 29.14. Coverage Reduction
  - when `rasterizationSamples` is greater than the fb samples

## Chapter 30. The Framebuffer

## Chapter 31. Dispatching Commands

## Chapter 32. Device-Generated Commands

## Chapter 33. Sparse Resources

- 33.1. Sparse Resource Features
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
  - `sparseBinding` feature
    - resouces can be bound at sparse block (i.e., gpu page) granularity
    - the entire resource must be bound to memory before use
    - if a format is supported, `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT` (but
      not `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT`) does not affect the result
      for both `VK_IMAGE_TILING_LINEAR` and `VK_IMAGE_TILING_OPTIMAL`
  - sparse residency features
    - resources do not have to be completely bound to memory before use
    - there are separate features for buffers, 2d images, 3d images, and msaa
      support
    - A sparse image created using `VK_IMAGE_CREATE_SPARSE_RESIDENCY_BIT`
      supports all non-compressed color formats with power-of-two element size
      that non-sparse usage supports
    - Additional formats may also be supported and can be queried via
      `vkGetPhysicalDeviceSparseImageFormatProperties`
    - `VK_IMAGE_TILING_LINEAR` tiling is not supported

## Chapter 34. Window System Integration (WSI)

- 34.1. WSI Platform
  - each platform-specific extension is an instance extension
- 34.2. WSI Surface
  - `VK_KHR_surface` provides `VkSurfaceKHR` to abstract platform
    surfaces/windows
  - `VK_KHR_android_surface` provides `vkCreateAndroidSurfaceKHR` to create a
    surface from a `struct ANativeWindow`
    - `currentExtent` is the window size and images are scaled to
      `currentExtent`
  - `VK_KHR_wayland_surface` provides `vkCreateWaylandSurfaceKHR` to create a
    surface from a `struct wl_surface`
    - `currentExtent` is always `UINT32_MAX` and `imageExtent` determines the
      window size
    - `VK_PRESENT_MODE_MAILBOX_KHR` is always supported
    - `wl_surface.attach`, `wl_surface.damage`, and `wl_surface.commit` must
      be sent and must only be sent by `vkQueuePresentKHR`
    - a separate `wl_event_queue` must be used
  - `VK_KHR_win32_surface` provides `vkCreateWin32SurfaceKHR` to create a
    surface from a `HWND`
    - `currentExtent` is the window size and can be 0
  - `VK_KHR_xcb_surface` provides `vkCreateXcbSurfaceKHR` to create a
    surface from a `xcb_window_t`
    - `currentExtent` is the window size and can be 0
  - `VK_EXT_metal_surface` provides `vkCreateMetalSurfaceEXT` to create a surface
    from a `CAMetalLayer`
- 34.3. Presenting Directly to Display Devices
  - `VK_KHR_display` provides
    - `vkGetPhysicalDeviceDisplayProperties2KHR` to enumerate `VkDisplayKHR`
      - on drm, each `drmModeConnectorPtr` of a gpu device corresponds to a
        `VkDisplayKHR`
    - `vkGetPhysicalDeviceDisplayPlaneProperties2KHR` to enumerate planes
      - on drm, mesa assumes each `drmModeConnectorPtr` has one plane
    - `vkGetDisplayPlaneSupportedDisplaysKHR` to query which planes are usable
      on which displays
    - `vkGetDisplayModeProperties2KHR` to enumerate `VkDisplayModeKHR`
      - on drm, each `drmModeModeInfoPtr` of a `drmModeConnectorPtr`
        corresponds to a `VkDisplayModeKHR`
    - `vkCreateDisplayModeKHR` to create new `VkDisplayModeKHR`
      - on drm, this returns an existing mode if any.  No new mode can be
        created.
    - `vkGetDisplayPlaneCapabilities2KHR` to query the caps of a plane/mode combo
    - finally, `vkCreateDisplayPlaneSurfaceKHR` to create a surface from a
      plane/mode combo
  - `VK_EXT_direct_mode_display` is for vr
    - `VK_EXT_acquire_drm_display` provides
      - `vkAcquireDrmDisplayEXT` to acquire a connector
        - on wayland, the idea is to use `wp_drm_lease_device_v1` to lease the
          master fd and to pass the master fd to vulkan using this extension
      - `vkGetDrmDisplayEXT` to query the `VkDisplayKHR` associated with a connector
  - `VK_EXT_display_control` provides `vkDisplayPowerControlEXT` to change the
    power state (off, on, suspend) of a display
  - `VK_EXT_headless_surface` provides `vkCreateHeadlessSurfaceEXT` to create
    a surface from thin air
    - it does not depend on `VK_KHR_display`
- 34.4. Querying for WSI Support
  - `VK_KHR_surface` provides `vkGetPhysicalDeviceSurfaceSupportKHR` to check
    if a queue family / surface combo is supported
  - `VK_KHR_wayland_surface` provides
    `vkGetPhysicalDeviceWaylandPresentationSupportKHR` to check if a queue
    family / `wl_display` combo is supported
  - `VK_KHR_win32_surface` provides
    `vkGetPhysicalDeviceWin32PresentationSupportKHR` to check if a queue
    family is supported
  - `VK_KHR_xcb_surface` provides
    `vkGetPhysicalDeviceXcbPresentationSupportKHR` to check if a queu family /
    `xcb_visualid_t` combo is supported
- 34.5. Surface Queries
  - `vkGetPhysicalDeviceSurfaceCapabilities2KHR` for surface caps
  - `vkGetPhysicalDeviceSurfaceFormats2KHR` for surface formats
  - `vkGetPhysicalDeviceSurfacePresentModesKHR` for surface present modes
- 34.7. Device Group Queries
  - `vkGetDeviceGroupPresentCapabilitiesKHR`
  - `vkGetDeviceGroupSurfacePresentModesKHR`
  - `vkGetPhysicalDevicePresentRectanglesKHR`
- 34.9. Present Wait
  - by chaining `VkPresentIdKHR` to `vkQueuePresentKHR`, `vkWaitForPresentKHR`
    can be used to wait for a present
- 34.10. WSI Swapchain
  - `VK_KHR_swapchain` (and etc.) provides
    - `vkCreateSwapchainKHR`
    - `vkDestroySwapchainKHR`
    - `vkGetSwapchainImagesKHR`
    - `vkAcquireNextImage2KHR`
    - `vkQueuePresentKHR`
    - `vkReleaseSwapchainImagesEXT`
  - `VK_KHR_shared_presentable_image` provides `vkGetSwapchainStatusKHR`
  - `VK_EXT_display_control` provides `vkGetSwapchainCounterEXT`
  - `VK_KHR_display_swapchain` provides `vkCreateSharedSwapchainsKHR`
  - `VK_KHR_present_wait` provides `vkWaitForPresentKHR`
- 34.11. Hdr Metadata
  - `VK_EXT_hdr_metadata` provides `vkSetHdrMetadataEXT` to set the hdr
    metadata for following presents

## Chapter 35. Deferred Host Operations

## Chapter 36. Private Data

## Chapter 37. Acceleration Structures

## Chapter 38. Micromap

## Chapter 39. Ray Traversal

## Chapter 40. Ray Tracing

## Chapter 41. Memory Decompression

## Chapter 42. Video Coding

## Chapter 43. Optical Flow

## Chapter 44. Execution Graphs

## Chapter 45. Low Latency 2

## Chapter 46. Extending Vulkan

- 46.1. Instance and Device Functionality
  - Commands that enumerate instance properties, or that accept a `VkInstance`
    object as a parameter, are considered instance-level functionality.
  - Commands that dispatch from a `VkDevice` object or a child object of a
    `VkDevice`, or take any of them as a parameter, are considered
    device-level functionality.
  - Commands that dispatch from `VkPhysicalDevice`, or accept a
    `VkPhysicalDevice` object as a parameter, are considered either
    instance-level or device-level functionality depending if the
    functionality is specified by an instance extension or device extension
    respectively.
  - Additionally, commands that enumerate physical device properties are
    considered device-level functionality.
- 46.2. Core Versions
  - The Vulkan version number comprises four parts indicating the `variant`,
    `major`, `minor` and `patch` version of the Vulkan API Specification.
  - The version of instance-level functionality can be queried by calling
    `vkEnumerateInstanceVersion`.
  - The version of device-level functionality is returned in
    `VkPhysicalDeviceProperties::apiVersion`
- 46.3. Layers
  - `vkEnumerateInstanceLayerProperties` enumerates instance layers
  - `vkEnumerateDeviceLayerProperties` has been deprecated
- 46.4. Extensions
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
- 46.6. Compatibility Guarantees (Informative)
  - extension
    - promotion: incorporated into core and absorbed by another extension
    - deprecation: no longer relevant
    - obsoletion: fundamentally incompatible with core or a better extension
    - aliases: to avoid duplicating documentation
    - special use: not recommended for general use
      - `cadsupport`
      - `d3demulation`
      - `devtools`
      - `debugging`
      - `glemulation`

## Chapter 47. Features

- Shader Data Type Widths
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

## Chapter 48. Limits

## Chapter 49. Formats

- 49.1. Format Definition
  - a list of all formats
  - 49.1.1. Compatible Formats of Planes of Multi-Planar Formats
    - a `_2PLANE` format has 2 format planes
    - a `_3PLANE` format has 3 format planes
    - `_420` has reduced width/height after the first plane
    - `_422` has reduced with after the first plane
    - Table 58. Plane Format Compatibility Table
  - 49.1.2. Multi-planar Format Image Aspect
  - 49.1.3. Packed Formats
    - formats with `_PACKnn` suffix are packed formats, and the naming
      convention is different
    - formats with `_mPACKnn` suffix are non-packed formats, and the naming
      convention does not change
      - except that each of the `m` compoents is considered packed
  - 49.1.4. Identification of Formats
    - `VK_FORMAT_{component-format|compression-scheme}_{numeric-format}`
    - the names can be followed by suffices
      - `_PACKnn`
      - `_mPACKnn`
      - `_BLOCK`
    - For multi-planar images
      - the components in separate planes are separated by underscores
      - the number of planes is indicated by the addition of a `_2PLANE` or
        `_3PLANE` suffix
      - `_444` indicates that all three planes of a three-planar image are
        the same size.
      - `_420` indicates that planes other than the first are reduced in
        size by a factor of two both horizontally and vertically,
      - `_422` indicates that planes other than the first are reduced in
        size by a factor of two horizontally or that the R and B values
        appear at half the horizontal frequency of the G values,
        - e.g., `VK_FORMAT_G8B8G8R8_422_UNORM` is non-planar and the R/B
          values appear at half the horizontal frequency
  - 49.1.5. Representation and Texel Block Size
    - Color formats must be represented in memory in exactly the form
      indicated by the format’s name.
    - Each format has a texel block size, the number of bytes used to store
      one texel block.
    - The representation of non-packed formats is that the first component
      specified in the name of the format is in the lowest memory addresses
      and the last component specified is in the highest memory addresses.
      - These include `_2PACK16` and `_4PACK16`, which are 2- and 4-comopnent
        non-packed formats.
    - Packed formats store multiple components within one underlying type. The
      bit representation is that the first component specified in the name of
      the format is in the most-significant bits and the last component
      specified is in the least-significant bits of the underlying type.
      - these are `_PACK8`, `_PACK16`, and `_PACK32` formats
    - Table 61. Byte mappings for non-packed/compressed color formats
    - Table 62. Bit mappings for packed 8-bit formats
    - Table 63. Bit mappings for packed 16-bit formats
    - Table 64. Bit mappings for packed 32-bit formats
  - 49.1.6. Depth/Stencil Formats
    - Depth/stencil formats are considered opaque and need not be stored in
      the exact number of bits per texel or component ordering indicated by
      the format enum.
    - However, implementations must not substitute a different depth or
      stencil precision than is described in the format (e.g. D16 must not be
      implemented as D24 or D32).
  - 49.1.7. Format Compatibility Classes
    - Compatible Formats
      - Uncompressed color formats are compatible with each other if they
        occupy the same number of bits per texel block as long as neither or
        both are alpha formats
      - Compressed color formats are compatible with each other if the only
        difference between them is the numeric format of the uncompressed
        pixels
      - Each depth/stencil format is only compatible with itself
    - Size Compatibility
      - Color formats with the same texel block size are considered
        size-compatible as long as neither or both are alpha formats
    - Table 65. Compatible Formats
- 49.2. Format Properties
  - `vkGetPhysicalDeviceFormatProperties2` queries format features
    - `VkFormatFeatureFlagBits` lists all possible features
      - some are specific to buffers and some are specific to images
    - `bufferFeatures` is for buffers
    - `linearTilingFeatures` is for images with `VK_IMAGE_TILING_LINEAR`
    - `optimalTilingFeatures` is for images with `VK_IMAGE_TILING_OPTIMAL`
    - `drmFormatModifierTilingFeatures` is for images with
      `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT`
      - different modifiers have different `drmFormatModifierTilingFeatures`
        and `drmFormatModifierPlaneCount`
- 49.3. Required Format Support
  - unless otherwise noted, the required format features must be supported for
    - every `VkImageType` (including arrayed and cube variants)
    - all `VkImageCreateFlags` values
  - when `VK_FORMAT_FEATURE_SAMPLED_IMAGE_BIT` is supported, these must also
    be supported
    - `VK_FORMAT_FEATURE_TRANSFER_SRC_BIT`
    - `VK_FORMAT_FEATURE_TRANSFER_DST_BIT`
    - `VK_FORMAT_FEATURE_2_HOST_IMAGE_TRANSFER_BIT_EXT` (for linear and
      optimal tilings)
  - the mandatory feature bits apply to `optimalTilingFeatures` and
    `bufferFeatures`
    - they do not apply to `linearTilingFeatures` or
      `drmFormatModifierTilingFeatures` according to
      - Table 66. Key for format feature tables
      - Table 67. Feature bits in optimalTilingFeatures
      - Table 68. Feature bits in bufferFeatures
  - Mandatory format support
    - Table 69. Mandatory format support: sub-byte components
    - Table 70. Mandatory format support: 1-3 byte-sized components
    - Table 71. Mandatory format support: 4 byte-sized components
    - Table 72. Mandatory format support: 10- and 12-bit components
    - Table 73. Mandatory format support: 16-bit components
    - Table 74. Mandatory format support: 32-bit components
    - Table 75. Mandatory format support: 64-bit/uneven components
    - Table 76. Mandatory format support: depth/stencil with VkImageType
      `VK_IMAGE_TYPE_2D`
    - one of
      - Table 77. Mandatory format support: BC compressed formats with
        VkImageType `VK_IMAGE_TYPE_2D` and `VK_IMAGE_TYPE_3D`
      - Table 78. Mandatory format support: ETC2 and EAC compressed formats
        with VkImageType `VK_IMAGE_TYPE_2D`
      - Table 79. Mandatory format support: ASTC LDR compressed formats with
        VkImageType `VK_IMAGE_TYPE_2D`
  - multi-planar (`nPLANE`) or sub-sampled (`422` or `420`) formats
    - when a view is created with `VK_IMAGE_ASPECT_PLANE_n_BIT`, the view
      format is the plane format
    - when a view is created with `VK_IMAGE_ASPECT_COLOR_BIT`, the view format
      is the image format and requires `samplerYcbcrConversion`
      - Table 80. Formats requiring sampler Y′CBCR conversion for
        `VK_IMAGE_ASPECT_COLOR_BIT` image views

## Chapter 50. Additional Capabilities

- 50.1. Additional Image Capabilities
  - an implementation can return `VK_ERROR_FORMAT_NOT_SUPPORTED` for any
    combination except for
    - those required by Required Format Support
    - if usage1/flags1 is supported, then usage2/flags2 must be supported if
      it is a subset of usage1/flags1
    - if usage includes `VK_IMAGE_USAGE_SAMPLED_BIT` and flags is non-sparse,
      `VK_IMAGE_USAGE_HOST_TRANSFER_BIT_EXT` must not affect the result
  - `VkImageFormatProperties`
    - `maxExtent` must be at least the same as specified in
      `VkPhysicalDeviceLimits`
    - `maxMipLevels` must support complete mipmaps unless
      - tiling is not `VK_IMAGE_TILING_OPTIMAL`,
      - the image is external, or
      - the format requires ycbcr conversion
    - `maxArrayLayers` must support array image unless
      - tiling is `VK_IMAGE_TILING_LINEAR`
      - image type is `VK_IMAGE_TYPE_3D`, or
      - the format requires ycbcr conversion

## Chapter 51. Debugging

- 51.1. Debug Utilities
  - `VK_EXT_debug_utils`
    - it is promoted from `VK_EXT_debug_marker`
    - it deprecates `VK_EXT_debug_report`
- 51.5. Active Tooling Information
  - `VK_EXT_tooling_info`
    - implemented by tools and layers to report themselves

## Appendix A: Vulkan Environment for SPIR-V

- Validation Rules Within a Module
  - `VK_KHR_shader_float_controls`
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
- Precision and Operation of SPIR-V Instructions
  - Precision of Individual Operations
    - Correctly Rounded
      - Operations described as "correctly rounded" will return the infinitely
        precise result, x, rounded so as to be representable in
        floating-point. The rounding mode is not specified, unless the entry
        point is declared with the RoundingModeRTE or the RoundingModeRTZ
        Execution Mode. These execution modes affect only correctly rounded
        SPIR-V instructions. These execution modes do not affect
        OpQuantizeToF16. If the rounding mode is not specified then this
        rounding is implementation specific, subject to the following rules.
        If x is exactly representable then x will be returned. Otherwise,
        either the floating-point value closest to and no less than x or the
        value closest to and no greater than x will be returned.

## Appendix B: Memory Model

- Availability, Visibility, and Domain Operations
  - when the first access scope includes `VK_ACCESS_HOST_WRITE_BIT`, the command
    includes a memory domain operation from the host domain to the device
    domain; when the second access scope include `VK_ACCESS_HOST_READ_BIT` or
    `VK_ACCESS_HOST_WRITE_BIT`, it includes a memory domain operation from the
    device domain to the host domain
  - `vkFlushMappedMemoryRanges`s includes an availability operation
  - `vkInvalidateMappedMemoryRanges` includes an visibility operation
  - `vkQueueSubmit` includes both a domain operation (from the host to the device)
    and a visibility operation

## Appendix C: Compressed Image Formats

## Appendix D: Core Revisions (Informative)

- Version 1.3
  - 23 promoted extensions
    - `VK_KHR_dynamic_rendering`
    - `VK_KHR_synchronization2`
- Version 1.2
  - 24 promoted extensions
    - `VK_KHR_buffer_device_address` for bindless resources
    - `VK_KHR_draw_indirect_count` for GL compat
    - `VK_KHR_separate_depth_stencil_layouts` for D3D compat
    - `VK_KHR_spirv_1_4` for HLSL compat
    - `VK_KHR_timeline_semaphore`
    - `VK_KHR_uniform_buffer_standard_layout` for HLSL compat
    - `VK_EXT_descriptor_indexing` for bindless resources
    - `VK_EXT_scalar_block_layout` for HLSL compat
    - `VK_EXT_separate_stencil_usage` for D3D compat
- Version 1.1
  - some of the new features
    - protected content
    - subgroup operations
    - `vkEnumerateInstanceVersion`
  - 23 promoted extensions
    - `VK_KHR_16bit_storage`
    - `VK_KHR_device_group` for multi-gpu
    - `VK_KHR_external_fence`
    - `VK_KHR_external_memory`
    - `VK_KHR_external_semaphore`
    - `VK_KHR_multiview` for vr
    - `VK_KHR_relaxed_block_layout` for HLSL compat
    - `VK_KHR_sampler_ycbcr_conversion`
    - `VK_KHR_variable_pointers` for advanced compute

## Appendix E: Layers & Extensions (Informative)

## Appendix F: Vulkan Roadmap Milestones

- Roadmap 2022
  - requires Vulkan 1.3
  - requires some optional features, noticeably
    - `samplerYcbcrConversion`
    - `descriptorIndexing`
  - requires increased limits

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
