Gallium Panfrost
================

## Source Files

- screen
  - `pan_screen.c` provides pipe screen callbacks
  - `pan_device.c` provides a wrapper for kmod
    - a `panfrost_device` consists of
      - a `pan_kmod_dev`
      - a `pan_kmod_vm`
      - a bo cache
      - a growable tiler heap before v10
      - more
  - `pan_bo.c` provides bo ops
    - a `panfrost_bo` wraps a `pan_kmod_bo`
    - it always uses the bo cache if allowed
  - `pan_fence.c` provides fence-related callbacks
    - a `pipe_fence_handle` consists of a syncobj
- context
  - `pan_context.c` provides pipe context callbacks
    - `panfrost_clear` uses the blitter
    - `panfrost_flush` flushes all batches
    - state callbacks update ctx states with dirty tracking
    - action and cso callbacks are providied by `pan_cmdstream.c`
- resource
  - `pan_resource.c` provides resource-related pipe screen and context
    callbacks
    - the screen callbacks are for alloc/free/import/export/query
    - the context callback are for map/unmap/blit/clear
- shader
  - `pan_shader.c` provides shader-related pipe context callbacks
    - there are vs, fs, and cs
  - `pan_disk_cache.c` is a wrapper for `disk_cache`
  - `pan_nir_remove_fragcolor_stores.c` provides an fs pass to remove color
    outputs that are beyond rt count
  - `pan_nir_lower_res_indices.c` provides a pass to select a resource table for
    each resource access
    - vs inputs use `PAN_TABLE_ATTRIBUTE`
    - ubos use `PAN_TABLE_UBO`
    - ssbos use `PAN_TABLE_SSBO`
    - images use `PAN_TABLE_IMAGE`
    - textures use `PAN_TABLE_TEXTURE`
    - samplers use `PAN_TABLE_SAMPLER`
  - `pan_nir_lower_sysvals.c` provides a pass to lower sysvals to ubo loads
    - i guess `bi_opt_push_ubo` will lower ubo loads to push consts
- command stream
  - `pan_cmdstream.c` provides action and cso pipe context callbacks
    - cso callbacks creates descriptors either on cpu or gpu memory
    - `panfrost_launch_grid` emits the compute state and job cmds to the batch
    - `panfrost_draw_vbo` emits the gfx state and job cmds to the batch
  - `pan_jm.c` is the jm backend for cmdstream
  - `pan_csf.c` is the csf backend for cmdstream
  - `pan_fb_preload.c` is shared by jm and csf backends
  - `pan_job.c` defines a `panfrost_batch` which is similar to a cmdbuf in vk
- helpers
  - `pan_blit.c` provides a wrapper for `util_blitter_blit`
  - `pan_mempool.c` provides a bo suballocator
  - `pan_helpers.c` provides misc helpers
  - `pan_afbc_cso.c` provides afbc shaders
    - they are compute shaders to convert AFBC-sprase to AFBC-packed

## Life of a Draw

- `panfrost_draw_vbo`
  - `prepare_draw`
    - `panfrost_get_batch_for_fbo` returns a batch
      - `panfrost_set_framebuffer_state` resets `ctx->batch` on fb change
      - `panfrost_get_batch` returns a compatible batch or a new batch
      - iow,
        - each batch corresponds to a different render pass
        - we allow up to `PAN_MAX_BATCHES` different render passes before
          internal flushing
  - `panfrost_update_state_3d`
    - `panfrost_emit_depth_stencil` allocs from `batch->pool` and inits
      `MALI_DEPTH_STENCIL` desc
    - `panfrost_emit_blend_valhall` allocs from `batch->pool` and inits
      `MALI_BLEND` descs
      - `panfrost_get_blend_shaders` creates and uploads blend shaders
    - `panfrost_emit_vertex_data` uploads `MALI_ATTRIBUTE` descs to
      `batch->pool`
    - `panfrost_emit_vertex_buffers` allocs from `batch->pool` and inits
      `MALI_BUFFER` descs
  - `panfrost_update_shader_state` is called for both vs and fs
    - `panfrost_emit_texture_descriptors` allocs from `batch->pool` and inits
      `MALI_TEXTURE` descs
    - `panfrost_emit_sampler_descriptors` allocs from `batch->pool` and inits
      `MALI_SAMPLER` descs
    - `panfrost_emit_compute_shader_meta` gets the resident shader desc
    - `panfrost_emit_images` allocs from `batch->pool` and inits
      `MALI_TEXTURE` descs
    - `panfrost_emit_ssbos` allocs from `batch->pool` and inits
      `MALI_BUFFER` descs
    - `panfrost_emit_const_buf` allocs from `batch->pool` and inits
      `MALI_BUFFER` descs
  - `JOBX(launch_draw)`
    - `GENX(csf_launch_draw)`
      - `csf_emit_draw_state` emits cmds to update various state regs
      - `csf_emit_draw_id_register` emits cmd to update draw id state reg
      - it emits cmds to update draw regs
      - `cs_run_idvs` emits `MALI_CS_RUN_IDVS`
    - `GENX(jm_launch_draw)`
      - `jm_emit_malloc_vertex_job` inits a `MALI_MALLOC_VERTEX_JOB`
      - `pan_jc_add_job` adds the job to the `vtc_jc` job chain
- `panfrost_batch_submit` submits a batch
  - each batch corresponds to a different render pass
  - it can be called from user flush (`panfrost_flush`) or internal flush
  - `panfrost_batch_to_fb_info` converts fb state to `pan_fb_info`
  - `panfrost_emit_tile_map` is for v5
  - `screen->vtbl.submit_batch` is `submit_batch`
    - `JOBX(prepare_tiler)`
      - `GENX(csf_prepare_tiler)` inits `MALI_TILER_CONTEXT`
      - `GENX(jm_prepare_tiler)` is nop
    - `JOBX(preload_fb)`
      - both `GENX(csf_preload_fb)` and `GENX(jm_preload_fb)` call the common
        `GENX(pan_preload_fb)`
    - `init_polygon_list` is for v5
    - `emit_tls` calls `GENX(pan_emit_tls)`
    - `emit_fbd`
      - both `GENX(csf_emit_fbds)` and `GENX(jm_emit_fbds)` calls the common
        `GENX(pan_emit_fbd)`
    - `emit_fragment_job`
      - `GENX(csf_emit_fragment_job)`
        - `cs_finish_tiling` emits `MALI_FINISH_TILING` to wait for tiling
        - it emits cmds to update various state regs
        - `cs_run_fragment` emits `MALI_CS_RUN_FRAGMENT`
        - `cs_finish_fragment` emits `MALI_FINISH_FRAGMENT` to wait for
          fragment
      - `GENX(jm_emit_fragment_job)` allocs from `batch->pool` and inits a
        `MALI_FRAGMENT_JOB` desc
    - `JOBX(submit_batch)`
      - `GENX(csf_submit_batch)`
        - `csf_emit_batch_end` ends the batch
        - `csf_prepare_qsubmit` preps a single `drm_panthor_queue_submit`
        - `csf_submit_gsubmit` submits
      - `GENX(jm_submit_batch)`
        - `jm_submit_jc` submits `vtc_jc` job chain
        - `jm_submit_jc` submits `frag` job chain with `PANFROST_JD_REQ_FS`
