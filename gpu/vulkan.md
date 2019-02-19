Vulkan
======

# Example

 - create a window
   - `xcb_connect` to create a connection and find the root screen
   - `xcb_create_window` and `xcb_map_window`
 - initialize vulkan
   - `vkCreateInstance`
   - `vkEnumeratePhysicalDevices`
   - `vkGetPhysicalDeviceProperties`
   - `vkGetPhysicalDeviceFeatures`
   - `vkGetPhysicalDeviceMemoryProperties`
   - `vkGetPhysicalDeviceQueueFamilyProperties`
   - `vkGetPhysicalDeviceFormatProperties`
   - `vkEnumerateDeviceExtensionProperties`
 - initialize a logical device
   - `vkCreateDevice`
   - `vkCreateCommandPool`
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

# Descriptor Sets

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

 - A pipeline layout consists of multiple descriptor set layouts and push
   constants.
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
