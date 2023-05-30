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
