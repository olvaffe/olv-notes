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
