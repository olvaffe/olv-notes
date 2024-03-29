Mesa Vulkan Runtime
===================

## Render Pass

- the runtime has a default render pass implementation that is built upon
  dynamic rendering
  - dynamic rendering still has load/store/resolve ops, but does not have the
    render passes, subpasses, subpass dependencies
  - this default implementation must insert barriers implied by render passes
- `vk_common_CreateRenderPass2`
  - allocs a `vk_render_pass` and multiple `vk_render_pass_attachment`,
    `vk_subpass`, `vk_subpass_dependency`, and `vk_subpass_attachment`,
  - parses vulkan structs into vk runtime structs
  - if there is a feedback loop between attachments in a subpass, forces the
    attachment layout to
    `VK_IMAGE_LAYOUT_ATTACHMENT_FEEDBACK_LOOP_OPTIMAL_EXT`
- `vk_common_DestroyRenderPass` frees `vk_render_pass`
- `vk_common_GetRenderAreaGranularity` returns 1x1
- `vk_common_CmdBeginRenderPass2`
  - updates states of `vk_command_buffer`
  - calls `begin_subpass`
- `vk_common_CmdNextSubpass2`
  - calls `end_subpass`
  - updates `subpass_idx` of `vk_command_buffer`
  - calls `begin_subpass`
- `vk_common_CmdEndRenderPass2`
  - calls `end_subpass`
  - calls `CmdPipelineBarrier2` to transition to the final layouts
  - updates states of `vk_command_buffer`
- `begin_subpass`
  - translates `vk_subpass_attachment` to `VkRenderingAttachmentInfo`
  - updates states of `vk_command_buffer`
  - calls `CmdPipelineBarrier2` to insert a memory barrier if there is any
    `vk_subpass_dependency` or to insert image barriers to transition to the
    initial layouts
    - there is an optimization using
      `VkRenderingAttachmentInitialLayoutInfoMESA` instead when the attachment
      is the entire image and the load op is `VK_ATTACHMENT_LOAD_OP_CLEAR`
    - it allows anv to hit a fast path called `will_full_fast_clear`
  - calls `load_attachment` that calls `CmdBeginRendering`/`CmdEndRendering`
    if an att is `VK_ATTACHMENT_LOAD_OP_CLEAR`
  - calls `CmdBeginRendering`
- `end_subpass`
  - calls `CmdEndRendering` to end rendering
  - calls `CmdPipelineBarrier2` to insert a memory barrier if there is any
    `vk_subpass_dependency` whose dst subpass is `VK_SUBPASS_EXTERNAL`

## Graphics Pipeline State

- `vk_graphics_pipeline_state_fill` flattens `VkGraphicsPipelineCreateInfo` to
  `vk_graphics_pipeline_state`
- `vk_get_dynamic_graphics_states` turns `VkPipelineDynamicStateCreateInfo`
  into a bitmask of `enum mesa_vk_dynamic_graphics_state`
- pipeline libraries
  - reminder
    - `VK_PIPELINE_CREATE_LIBRARY_BIT_KHR` means to create a pipeline library
      - i.e., a pipeline that is incomplete
    - `VkPipelineLibraryCreateInfoKHR` means to create a complete pipeline or
      a pipeline library from the specified pipeline libraries
    - `VkGraphicsPipelineLibraryCreateInfoEXT` specifies the graphics states
      included directly in `VkGraphipcsPipelineCreateInfo`
      - the complete pipeline or pipeline library being created will have
        graphipcs states directly from `VkGraphipcsPipelineCreateInfo` or (if
        any) indirectly from `VkPipelineLibraryCreateInfoKHR`
  - `allowed_stages` is
    - if creating a regular pipeline, it's
      - `VK_SHADER_STAGE_ALL_GRAPHICS`
      - `VK_SHADER_STAGE_TASK_BIT_EXT`
      - `VK_SHADER_STAGE_MESH_BIT_EXT`
    - if creating a graphics pipeline library, it depends on
      - `VK_GRAPHICS_PIPELINE_LIBRARY_PRE_RASTERIZATION_SHADERS_BIT_EXT`
      - `VK_GRAPHICS_PIPELINE_LIBRARY_FRAGMENT_SHADER_BIT_EXT`
    - otherwise, zero
- `FOREACH_STATE_GROUP` macro
  - it takes another macro as input
  - it invokes the other macro with all `mesa_vk_graphics_state_groups` one by
    one
    - the first parameter is `mesa_vk_graphics_state_groups`
    - the second parameter is the type of state group
    - the third parameter is the name of state group in
      `vk_graphics_pipeline_state`
- `lib`
  - if gpl, it's `VkGraphicsPipelineLibraryCreateInfoEXT::flags`
  - else, it's 0 or all
- `needs` is a bitmask of `enum mesa_vk_graphics_state_groups`
  - it specifies the graphics states to be flatten to
    `vk_graphics_pipeline_state`
  - it is initialized to graphipcs states directly included in
    `VkGraphicsPipelineCreateInfo`
  - `FOREACH_STATE_GROUP(FILTER_NEEDS)` filters out states that are already
    flattened previously
  - if a state group is fully dynamic, it is also removed from `needs`
    - `get_dynamic_state_groups` maps `mesa_vk_graphics_state_groups` to
      `mesa_vk_dynamic_graphics_state`
    - `fully_dynamic_state_groups` maps `mesa_vk_dynamic_graphics_state` to
      `mesa_vk_graphics_state_groups`
- `ma`
  - `FOREACH_STATE_GROUP(ENSURE_STATE_IF_NEEDED)` calls `vk_multialloc_add`
    for needed states
  - `all` can be provided as the storage to avoid multialloc
- `INFO_ALIAS` maps names in `VkGraphicsPipelineCreateInfo` to names in
  `vk_graphics_pipeline_state`
- `FOREACH_STATE_GROUP(INIT_STATE_IF_NEEDED)` initializes each needed
  graphics state groups

## `vk_sync_type`

- sync types provided by the runtime
  - `vk_drm_syncobj_get_type` uses native syncobj
  - `vk_sync_binary_get_type` provides binary sync over timeline sync
  - `vk_sync_timeline_get_type` provides timeline sync over binary sync
    - `vk_sync_timeline_type_validate` has requirements for the binary sync
  - `vk_sync_dummy_type` is fake and does not sync at all
    - it is used by WSI in some cases
- `vk_physical_device::supported_sync_types` is initialized during physical
  device enumeration
  - drivers usually call `vk_drm_syncobj_get_type` to get the sync type for
    native syncobj
    - `drmSyncobjCreate` and `DRM_SYNCOBJ_CREATE_SIGNALED` map to
      - `VK_SYNC_FEATURE_BINARY`
      - `VK_SYNC_FEATURE_GPU_WAIT`
      - `VK_SYNC_FEATURE_CPU_RESET`
      - `VK_SYNC_FEATURE_CPU_SIGNAL`
      - `VK_SYNC_FEATURE_WAIT_PENDING`
    - `drmSyncobjWait` and `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL` map to
      - `VK_SYNC_FEATURE_CPU_WAIT`
      - `VK_SYNC_FEATURE_WAIT_ANY`
    - `DRM_CAP_SYNCOBJ_TIMELINE` maps to `VK_SYNC_FEATURE_TIMELINE`
      - this is a relatively newer feature
  - `vk_sync_timeline_get_type` can wrap a binary sync type to provides
    emulated `VK_SYNC_FEATURE_TIMELINE` support
    - timeline can be fully emulated except for export/import
- `vk_device::timeline_mode` and `vk_device::submit_mode` are initialized in
  `vk_device_init`
  - `get_timeline_mode` checks `vk_physical_device::supported_sync_types`
    - if no timeline sync, `VK_DEVICE_TIMELINE_MODE_NONE`
    - if emulated timeline sync, `VK_DEVICE_TIMELINE_MODE_EMULATED`
    - if `VK_SYNC_FEATURE_WAIT_BEFORE_SIGNAL`, which is only supported by
      dozen, `VK_DEVICE_TIMELINE_MODE_NATIVE`
    - else `VK_DEVICE_TIMELINE_MODE_ASSISTED`
  - `vk_device::submit_mode` is based on `vk_device::timeline_mode`
    - the effective submit mode of a queue will be determined in
      `vk_queue_init`
- `get_fence_sync_type` determines the sync type for a `VkFence`
  - it returns the first sync type with
  - `VK_SYNC_FEATURE_BINARY`,
  - `VK_SYNC_FEATURE_CPU_WAIT`
  - `VK_SYNC_FEATURE_CPU_RESET`
- `get_semaphore_sync_type` determines the sync type for a `VkSemaphore`
  - for binary semaphores, it returns the first sync type with
    - `VK_SYNC_FEATURE_BINARY`
    - `VK_SYNC_FEATURE_GPU_WAIT`
  - for timeline semaphores, it returns the first sync type with
    - `VK_SYNC_FEATURE_TIMELINE`
    - `VK_SYNC_FEATURE_GPU_WAIT`
    - `VK_SYNC_FEATURE_CPU_WAIT`

## `vk_common_QueueSubmit2`

- `vk_queue_submit` submits each `VkSubmitInfo2` one by one
  - `VkSubmitInfo2` is converted to `vulkan_submit_info` first
  - `vulkan_submit_info` is parsed into `struct vk_queue_submit`
  - `struct vk_queue_submit` is submitted depending on the queue submit mode
- queue submit modes
  - `VK_QUEUE_SUBMIT_MODE_IMMEDIATE` calls `vk_queue_submit_final` immediately
    - this calls `queue->driver_submit` after pre-processing
    - timeline semaphores allow wait-before-submit and this mode normally does
      not work j
    - e.g., if submit A waits on a semaphore that is signaled by submit B on a
      different queue, the runtime may get submit A before submit B but the
      kernel requires submit B before submit A
  - `VK_QUEUE_SUBMIT_MODE_DEFERRED` calls `vk_queue_push_submit` and
    `vk_device_flush`
    - this uses `VK_SYNC_WAIT_PENDING` to check if all dependencies of a
      submit have become pending (submitted) before calling
      `vk_queue_submit_final`
  - `VK_QUEUE_SUBMIT_MODE_THREADED` calls `vk_queue_push_submit`
    - the queue thread uses `VK_SYNC_WAIT_PENDING` to wait all dependencies of
      a submit before calling `vk_queue_submit_final`
