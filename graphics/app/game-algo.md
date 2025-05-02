3D Techniques
=============

## Lighting

- for each fragment, we want to calculate the light perceived by the camera
- the perceived light is the result of
  - object color
  - ambient light
  - point lights
  - directional lights
  - more
- a simplest model gives
  - `objColor * (ambientColor + directionalColor * dot(objNormal, directionalLightDir))`

## Shadow Mapping

- first pass: render the scene from the light's point of view
  - depth only; no color attachment
  - vs only; no fs
  - a depth value indicates the distance from the light
  - this generates a shadow map
- second pass: render the scene from the camera's point of view
  - vs: calculate positions from the camera's and the light's point of view
  - fs: when the distance from the light is larger than the value stored in
    shadow map, the fragment is not lit by the light
- when there are multiple lights,
  - first pass is done for each light
  - second pass is done once, but all lights are added together in fs

## Deferred Shading

- the straighforward forward shading
  - has complexity of `O(number_of_lights * number_of_objects)`
  - lots of overdrawn when objects occlude each other
- deferred shading
  - has complexity of `O(number_of_lights * window_size)`
  - zero overdrawn
- geometry pass: render the scene to g-buffer
  - vs is the normal vs
  - fs is a special one that writes diffferent infos to different render
    targets
    - one RT for transformed position
    - one RT for transformed normal
    - one RT for object color
    - that is, instead of doing lighting using the infos, we write them out to
      different RTs
  - the collection of the render targets is called g-buffer (geometry buffer)
- lighting pass: render a window-size rectangle
  - vs: do nothing
  - fs: read lighting infos from g-buffer; lighting normally

## Model Rendering

- a 3D model consists of an array of sub-meshes
- each sub-mesh has
  - vertex attributes which are each an array of values
    - position
    - normal
    - texcoord
  - material
    - albedo texture (or constant color)
    - normal texture (or just "normal" vertex attribute)
    - metallic texture (or just a float value)
    - roughness texture (or just a float value)
    - more
  - optionally an array of indices (i.e., index buffer)
- to render a model,
  - switch vbo/ibo/textures/samplers for each sub-mesh
  - vs: transform position and normal
  - fs: sample material textures and shadow maps; apply physically-based
    lighting
- skinning and animation
  - TODO

## Terrain Rendering

- can be rendered like a model
  - highly detailed mesh
  - albedo texture
  - normal texture
- but it is better to do
  - a grid with height map
    - vs samples the height map to adjust position of the grid
  - albedo texture
  - normal texture
- designer usually go like
  - oh, my terrain is going to be 5km x 5km
  - my resolution is 50cm
  - I need 10000 x 10000 grid
- the textures cannot fit GPU memory
- tricks
  - divide terrain into tiles
  - LOD depending on how far a tile is from the camera
    - LOD for both the mesh and the textures
  - dynamic tile streaming and sparse textures
    - alternatively, use a splatmap

## Life of a Frame

- per-frame data
  - there is a VkSwapchainKHR with N images
  - there should be N FrameData that manage per-frame data
- acquire swapchain image for the next frame
  - allocate a VkSemaphore from the current FrameData
  - use the semaphore with vkAcquireNextImageKHR
- make the next frame current
  - vkWaitForFences on all fences in next FrameData
  - objects in the next FrameData can be reused after the wait
- render the current frame
  - first vkQueueSubmit waits the swapchain image acquire semaphore
  - last vkQueueSubmit allocates a VkSemaphore and a VkFence from the current
    FrameData and signals both
- present the current frame
  - vkQueuePresentKHR that waits the last rendering semaphore
- repeat
  - when the current FrameData is picked again, all fences including the last
    rendering fence will be waited and the current FrameData can be reused
  - hmm, something feels wrong with the swapchain image acquire semaphore

## Per-Frame Data

- examples of per-frame data, FrameData, are
  - swapchain image
  - accompanying depth image
  - fences and semaphores
  - command pools and command buffers
  - small buffers
  - descriptor pools and descriptor sets
- FrameData should
  - manage objects needed for frame rendering
  - objects have liftime of the frame
  - reuse as many objects as possible when the FrameData is picked again
  - lock-free even with multiple threads
- fence pool
  - a dynamic array of VkFence
  - when reuse, vkWaitForFences and vkResetFences
    - because of the wait, after the fence pool is reused, all other objects
      in FrameData are idle
- semaphore pool
  - a dynamic array of VkSemaphore
  - caller must wait semaphores allocated from the pool
    - such that they are unsignaled ultimately
  - when reuse, assume all semaphores have been waited
    - caller must wait for a fence paired with the pool
- command buffer pool
  - consists of 
    - a VkCommandPool
    - a dynamic array of primary VkCommandBuffer
    - a dynamic array of secondary VkCommandBuffer
  - when resue, vkResetCommandPool
  - there should be multiple command buffer pools
    - one for each queue family
    - one for each thread (to be lock-free)
- small buffer pool
  - consists of
    - a dynamic array of fixed-size (e.g., 256KB) VkBuffer
    - each allocation suballocates and does not exceed the fixed-size
  - when reuse, set internal offset counter to 0
  - there should be multiple small buffer pools
    - one for each supported VkBufferUsageFlags
    - one for each thread (to be lock-free)
- descriptor set pool
  - consists of
    - a dynamic array of VkDescriptorPool
    - each VkDescriptorPool can hold a fixed-number (e.g., 16) of VkDescriptorSet
  - when reuse, vkResetDescriptorPool
    - note that this invalidates VkDescriptorSet
    - alternatively, and more commonly, we can do nothing
      - for the same material across frames, its VkDescriptorSet is the same
      - we can cache VkDescriptorSet and use
	(VkDescriptorLayout, VkWriteDescriptorSet) as the key to look up the
	VkDescriptorSet
      - this treats the VkDescriptorSet as a per-material one
      - normally, only UBO varies per-frame.  By allowing the UBO binding of
        the VkDescriptorSet to vary, the VkDescriptorSet can be treated as a
        per-scene one
  - there should be multiple descriptor set pools
    - one for each VkDescriptorLayout
    - one for each thread (to be lock-free)

## Model Rendering

- a model is described by
  - a mesh
    - input attributes (pos, normal, texcoord)
    - raw input data
  - a material
    - parameters (roughness, color, etc.)
    - parameters can be in texture form when they vary pixel-by-pixel
- at loading time,
  - remember input attributes
    - input type (pos, normal, or texcoord)
    - input stride
    - input format (VkFormat)
    - input offset within stride
  - upload raw input data to VkBuffer
  - remember material parameters
    - for uploading as uniforms later
  - upload textures to VkImage/VkImageView/VkSampler
- also at loading time, or defer until draw time, create some pieces of
  VkGraphicsPipelineCreateInfo
  - it depends on how static these pieces, mostly shader modules, are
    - when a different graphics quality is chosen, we might choose different
      shader modules
    - it might be better to defer until draw time
  - VkPipelineShaderStageCreateInfo
    - given the material (and quality and scene), we know exactly which
      shaders to use
    - upload shaders as VkShaderModule
    - inspect shaders and remember
      - locations of input attributes
      - offsets/sizes of push constant ranges
      - sets/bindings of UBOs
      - sets/bindings of material textures
  - VkPipelineVertexInputStateCreateInfo
    - for each input attribute of vs
      - as the author of vs, we know the type (pos, normal, etc.)
      - by using compact location=, we can set binding to location
      - we can find input stride/format/offset from the model
    - these are enough to initialize VkPipelineVertexInputStateCreateInfo
    - these are enough for vkCmdBindVertexBuffers
  - VkPipelineLayout
    - for each uniform of all shader modules,
      - we know its set/binding/type
      - this is enough to initialize a VkDescriptorSetLayoutBinding
    - we can create a VkDescriptorLayout for each set
      - by grouping uniforms and VkDescriptorSetLayoutBinding by sets
    - push constan ranges are already known through shader module inspection
- at draw time,
  - after all states of VkGraphicsPipelineCreateInfo are known, we can create
    the VkPipeline
    - we definitely need a cache for VkPipeline
    - "all states" being the lookup key
  - we can create VkDescriptorSet easily when VkPipelineLayout (thus
    VkDescriptorSetLayout) known
    - we want a cache to minimize vkUpdateDescriptorSets
  - for each texture of all shader modules,
    - as the author of the shader, we know the corresponding texture in the
      model and the VkImageView/VkSampler created at loading time
    - with set, binding, VkImageView, and VkSampler known, we can initialize a
      VkWriteDescriptorSet
    - these are enough for vkUpdateDescriptorSet and vkCmdBindDescriptorSets
  - for each ubo of all shader modules,
    - we will allocate a VkBuffer (or pick a range of a huge VkBuffer)
    - we fill map it and fill in transformations, material parameters, etc.
    - the descripts will be updated together with the textures

## Life of a Scene

- per-scene data, SceneData
- load assets into SceneData
- render frames
- free SceneData
- repeat

## Per-Scene Data

- examples of per-scene data, SceneData, are
  - scene graph
  - buffers and buffer views
  - images and image views
  - samplers
  - shader modules, pipeline layouts, and pipelines
  - render passes
- given a mesh and its material,
  - we can pre-create its VkShaderModule at load time
  - we have to create its VkDescriptSetLayout, VkPipelineLayout and VkPipeline
    at draw time, when all states are known
  - to create its VkDescriptorSetLayout and VkPipelineLayout
    - parse VkShaderModule to know resource (name, set, binding, arraySize)
    - build a `name -> resource` mapping
      - this will be needed for VkWriteDescriptorSet and others
    - build a `set -> vector<resource>` mapping
      - for each set, create a VkDescriptorSetLayout
	- each resource needing a descriptor is converted to a
	  VkDescriptorSetLayoutBinding
    - create a VkPipelineLayout from the VkDescriptorSetLayout
  - to create its VkPipeline
    - loop over VS inputs to setup VkPipelineVertexInputStateCreateInfo and to
      vkCmdBindVertexBuffers
    - vkCreateGraphicsPipelines and vkCmdBindPipeline
    - vkCmdPushConstants
    - vkAllocateDescriptorSets, vkUpdateDescriptorSets and
      vkCmdBindDescriptorSets
- use an object cache for
  - VkShaderModule
  - VkPipelineLayout
  - VkDescriptorSetLayout
  - VkPipeline
  - VkDescriptorSet
  - VkRenderPass
  - VkFramebuffer

## glTF Viewer

- at init time, create a "render context"
  - a "render context" owns a VkSwapchainKHR
  - for each swapchain image, create a FrameData
  - a FrameData manages the swapchain image as well as per-frame data
- at load time, create a "scene graph"
  - for each sampler, create a VkSampler
  - for each image, create a VkImage, a VkDeviecMemory, and a VkImageView
    - use vkCmdCopyBufferToImage to upload to GPU
  - for each texture, create a struct to pair image/sampler
  - for each material, create a struct to store the PBR parameters and textures
  - for each mesh, create a struct to an array of sub-meshes
    - for each attribute of a sub-mesh, create a VkBuffer, a VkDeviceMemory,
      and a VkVertexInputAttributeDescription
    - if there is an index buffer, do the same
    - store the material
  - for each camera, create a struct to store the camera pos and etc
  - associate each mesh/camera with their node (for things such as local
    transformations)
- create a "render pipeline" from the "scene graph"
  - for each sub-mesh in a mesh, pre-create the VkShaderModules
- at draw time, invoke the "render pipeline"
  - vkBeginCommandBuffer
  - vkCreateRenderPass with cache
  - vkCreateFramebuffer with cache
  - vkCmdBeginRenderPass
  - sort sub-meshes using current camera
    - sort opaque sub-meshes front-to-back
    - sort transparent sub-meshes back-to-front
  - for each opaque sub-mesh,
    - allocate a VkBuffer for transform
    - bind the VkBuffer to (set=0, binding=1)
    - bind the VkBuffer to (set=0, binding=1)
    - PipelineLayout
      - inspect shader source code and get all resources (var name, set,
      	binding)
  - vkCmdEndRenderPass
  - vkEndCommandBuffer

## Object Cache

- there are objects that we want to cache
  - we want to cache objects that are persistent and are reused
    - especially those that are expensive to create
  - use hashmap to map their create infos to handles
- VkRenderPass
  - there are only a handful of them
  - they are constantly reused
- VkFramebuffer
  - similar to VkRenderPass, but swapchainDepth times more
  - also changes with swapchain resize
- VkShaderModule
  - there are some for each scene node type and quality settings
  - they are constantly reused to create VkPipeline
- VkPipelineLayout
  - they can be created with VkShaderModule reflection
  - they are constantly reused to create VkPipeline
  - they depend on VkDescriptorSetLayout
    - should be cached too
    - there are also immutable VkSampler if used
- VkPipeline
  - there are thousands of them
  - not all of them are constantly reused
  - but they are too expensive that we definitely should cache
  - there are dependencies
    - VkRenderPass
    - VkShaderModule
    - VkPipelineLayout
- VkDescriptorSet
  - they are not created from VkDevice, but allocated from VkDescriptorPool
  - doesn't need exact match, but want to minimize vkUpdateDescriptorSets
