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
 - A descriptor set is suballocated from the pool
 - A descriptor set layout is described as
   - binding X: N descriptors of certain type used by certain shader stages
   - there can be arbitrarily many bindings, using arbitrarily binding numbers
   - this is enough for impl to calculate the total number of HW descriptors
     required and to maps "descriptor #n at binding X" to "HW descriptor #m"

# Command Buffers

 - A command buffer consists of
   - a list of BOs, dynamically growing, with one chained to another
   - a vector of reloc entries (if resources may move on the impl)
   - a vector of binding table blocks
 - A command pool is almost a dummy object to reset/free a group of command
   buffers in one go
