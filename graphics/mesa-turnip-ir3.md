Mesa Turnip IR3
===============

## IR3 Public APIs

- drivers include `ir3_comiler.h`, `ir3_shader.h`, and `ir3_nir.h` to access
  the public APIs
- `tu_CreateDevice`
  - `ir3_compiler_create` creates the top-level `struct ir3_compiler`
  - turnip sets `disable_cache` and manages disk cache itself
- `tu_DestroyDevice`
  - `ir3_compiler_destroy` destroys the top-level `struct ir3_compiler`
- `tu_pipeline_builder_compile_shaders` or `tu_compute_pipeline_create`
  - `tu_spirv_to_nir` converts spirv to nir
    - `ir3_get_compiler_options` to get `nir_options`
    - `ir3_optimize_loop` applies generic optimizations
  - `tu_shader_create` creates `ir3_shader`
    - `ir3_nir_lower_io_to_temporaries` applies some std passes
    - `ir3_finalize_nir` applies more passes and the nir is ready
    - `ir3_shader_from_nir` creates `ir3_shader` to wrap nir
  - `ir3_shader_create_variant` created `ir3_shader_variant`
    - this is the most heavy function
  - `ir3_trim_constlen` checks variants do not exceed `constlen` limits
    - drivers need to re-create variants with `safe_constlen` if they exceed
      the limits
  - `tu_shader_destroy`
    - `ir3_shader_destroy` destroys `ir3_shader` now that turnip has
      `ir3_shader_variant`
- `tu_pipeline_builder_parse_*`
  - `ir3_const_state` gets the const register file layout
  - `ir3_shader_branchstack_hw`
  - `ir3_link_shaders` updates `ir3_shader_linkage` for vs and fs
  - `ir3_link_stream_out` updates `ir3_shader_linkage` for vs and so
  - `ir3_link_add` updates `ir3_shader_linkage` even more
  - `ir3_find_output_regid` is for linkage of outputs
  - `ir3_find_sysval_regid` is for linkage of sysvals
- misc
  - pipeline cache
    - `ir3_store_variant`
    - `ir3_retrieve_variant`
  - `ir3_shader_get_variant`
    - it is the same as `ir3_shader_create_variant`, except the memory is
      managed by `ir3_shader`

## IR3 Source Code Structure

- driver apis
  - `ir3_compiler.c`
    - `ir3_compiler_create`, top-level function for  use by drivers
    - `ir3_disk_cache.c` is disabled by turnip
  - `ir3_shader.c`
    - `ir3_shader_from_nir`
    - `ir3_shader_create_variant`
    - `ir3_shader_get_variant`
    - `ir3_shader_assemble`
    - `ir3_shader_disasm`
      - disabbembly for debug
    - `ir3_shader_outputs`, a3xx only
    - `ir3_link_stream_out` adds so linkage
- `ir3_shader_create_variant` complies nir
  - `ir3_nir.c` provides nir post-processing
    - `ir3_finalize_nir` has been called by drivers
    - `ir3_nir_post_finalize` is called once before creating variants
      - `ir3_nir_lower_load_barycentric_at_offset.c`
      - `ir3_nir_lower_load_barycentric_at_sample.c`
      - `ir3_nir_move_varying_inputs.c`
      - `ir3_nir_trig.py`
    - `ir3_nir_lower_variant` is called from `ir3_context_init` (which is
      called from `ir3_compile_shader_nir`) for each variant
      - `ir3_nir_lower_tess.c`
      - `ir3_nir_lower_wide_load_store.c`
      - `ir3_nir_lower_64b.c`
      - `ir3_nir_opt_preamble.c`
      - `ir3_nir_analyze_ubo_ranges.c`
      - `ir3_nir_lower_io_offsets.c`
  - `ir3_context.c` manages nir to ir3 translation
    - `ir3_context_init` applies NIR passes before the translation
      - `ir3_nir_lower_variant`
      - `ir3_nir_imul.py`
      - `ir3_nir_lower_tex_prefetch.c`
  - `ir3_compiler_nir.c` translates nir to ir3
    - `ir3_compile_shader_nir`
      - `ir3_context_init` is called to prepare nir for translation
      - `emit_instructions` translates `nir_instr` to `ir3_instruction`
        - `OPC_META_*` and `OPC_*_MACRO` are pseudo instructions that will be
          lowered from various places
      - `ir3.c` is for working with IR3
        - `ir3_a4xx.c` is a4xx backend
        - `ir3_a6xx.c` is a6xx backend
        - `ir3_image.c` is for ibo/ssbo related stuff
	- `ir3_delay.c` is used by `ir3_sched`, `ir3_postsched`, and
	  `ir3_legalize` for nop-delays
      - ir3 passes are applied
        - `ir3_remove_unreachable.c`
        - `ir3_array_to_ssa.c`
        - `ir3_cf.c`
        - `ir3_cp.c`
        - `ir3_cse.c`
        - `ir3_dce.c`
      - `ir3_sched.c` schedules instructions (to minimize nop-delays) pre-RA
      - `ir3_ra.c` performs register allocation and spilling
        - `ir3_dominance.c`
        - `ir3_liveness.c`
        - `ir3_merge_regs.c`
        - `ir3_spill.c`
        - `ir3_lower_spill.c`
        - `ir3_lower_parallelcopy.c`
      - `ir3_postsched.c` schedules instructions post-RA
      - `ir3_lower_subgroups.c` lowers  `OPC_MACRO_*` that are
      	subgroup-related
      - `ir3_legalize.c`
        - it also lowers `OPC_DSXPP_MACRO`
      - `ir3_legalize.c`
      - `collect_tex_prefetches` lowers `OPC_META_TEX_PREFETCH`
  - `assemble_variant` generates the native code
    - `ir3_shader_assemble` calls `isa_assemble` from `libir3encode`
- utils
  - `ir3_print.c` provides debug prints
  - `ir3_validate.c` and `ir3_ra_validate.c` are only enabled on debug build
    for validations
  - `disasm-a3xx.c` and `ir3_assembler.c` are only used by tools

## IR3 VS IOs

- spirv
  - `OpVariable`
  - `OpLoad`
  - `OpStore`
- nir
  - `decl_var`
  - `deref_var`
  - `load_deref`
  - `store_deref`
- `nir_lower_io`
  - `load_input`
  - `store_output`
- `ir3_compile_shader_nir`
  - `setup_input` lowers `nir_intrinsic_load_input` to `OPC_META_INPUT`
    - `ir3_legalize` will discard most meta instructions including
    	`OPC_META_INPUT`
  - `create_sysval_input`
  - `setup_output` lowers `nir_intrinsic_store_output` to nothing
- `ir3_shader_variant`
  - `outputs_count` and `outputs`
    - these are `gl_varying_slot`
  - `output_size` is used only when HS/DS/GS is enabled
  - `inputs_count` and `inputs`
    - these include `gl_vert_attrib` and `gl_system_value`
  - `input_size` is 0 because it is only for HS/DS/GS
  - `total_in`, `sysval_in`, and `varying_in` are set but are not used for
    VS
    - they are mostly used for fs before a6xx
  - `tu6_emit_vertex_input` uses `inputs` to emit `A6XX_VFD_DEST_CNTL_INSTR`
- when HS/DS/GS is enabled
  - vs uses `chsh` to jump to the next stage
  - `ir3_nir_lower_to_explicit_output` initializes `output_loc` and
    `output_size`
    - it uses `shader_io_get_unique_index` to map `gl_varying_slot` to byte
    	offsets
    - `output_size` is in dwords (e.g., 2 out's have 8 dwords)
    - `store_output` is lowered to `store_shared_ir3`
  - system values
    - `SYSTEM_VALUE_TCS_HEADER_IR3` is written to
    	`VARYING_SLOT_TCS_HEADER_IR3`
    - `SYSTEM_VALUE_REL_PATCH_ID_IR3` is written to
    	`VARYING_SLOT_REL_PATCH_ID_IR3`
    - `SYSTEM_VALUE_GS_HEADER_IR3` is written to
    	`VARYING_SLOT_GS_HEADER_IR3`
    - `SYSTEM_VALUE_PRIMITIVE_ID` is written to `VARYING_SLOT_PRIMITIVE_ID`

## IR3 HS/DS IOs

- spirv
  - `OpVariable`
  - `OpLoad`
  - `OpStore`
  - `AccessChain`
- nir
  - `decl_var`
  - `deref_var`
  - `load_deref`
  - `store_deref`
  - `deref_array`
- `nir_lower_io`
  - `load_per_vertex_input`
  - `store_per_vertex_output`
- ir3-specific tess lowering
  - `ir3_nir_lower_tess_ctrl`
    - `load_per_vertex_output` and `load_output` are lowered to
      `load_global_ir3`
    - `store_per_vertex_output` and `store_output` are lowered to
      `store_global_ir3`
    - these use a bo of size `TU_TESS_BO_SIZE`
  - `ir3_nir_lower_tess_eval`
    - `load_per_vertex_input` and `load_input` are lowered to
      `load_global_ir3`
  - `ir3_nir_lower_to_explicit_output`
    - `store_output` is lowered to `store_shared_ir3` and uses the shared
      (local) memory
    - `output_loc` and `output_size` describe the layout of the local memory
  - `ir3_nir_lower_to_explicit_input`
    - `load_per_vertex_input` is lowered to `load_shared_ir3`
  - VS
    - when followed by HS/DS/GS, use `ir3_nir_lower_to_explicit_output` to
      lower outputs to share (local) memory
    - otherwise, no special handling
  - HS is always followed by DS
    - use `ir3_nir_lower_tess_ctrl` to lower outputs to a `TU_TESS_BO_SIZE` bo
    - use `ir3_nir_lower_to_explicit_input` to lower inputs to share (local)
      memory
  - DS is always preceded by HS
    - use `ir3_nir_lower_tess_eval` to lower inputs to a `TU_TESS_BO_SIZE` bo
    - if followed by GS, also use `ir3_nir_lower_to_explicit_output` to lower
      outputs to share (local) memory
  - GS is always preceded by VS/HS/DS
    - use `ir3_nir_lower_to_explicit_input`
- HS `ir3_shader_variant`
  - `inputs_count` and `inputs`
    - `gl_varying_slot` has been lowered to local memory
    - these are system values such as `SYSTEM_VALUE_TCS_HEADER_IR3` and
      `SYSTEM_VALUE_REL_PATCH_ID_IR3`
  - `input_size` is set by `ir3_nir_lower_to_explicit_input` and is the max
    unique index plus 1
  - `outputs_count` and `outputs` are 0
  - `output_size` is set by `ir3_nir_lower_tess_ctrl`
- DS `ir3_shader_variant`
  - `inputs_count` and `inputs` are the same as in HS
  - `input_size` is by `ir3_nir_lower_tess_eval` and is the max unique index
    plus 1
  - with GS, `outputs_count` and `outputs` are 0.  `output_size` is set.
  - without GS, `outputs_count` and `outputs` are for fs.  `output_size` is 0.

## IR3 UBOs

- descriptors
  - `descriptor_size` returns `A6XX_TEX_CONST_DWORDS * 4`, 64 bytes, for UBOs
  - `write_ubo_descriptor` uses only 8 bytes
  - I guess this is because adreno bindless expects descriptors to be
    multiples of 64 bytes
- spirv-to-nir
  - use `nir_intrinsic_vulkan_resource_index` and
    `nir_intrinsic_load_vulkan_descriptor` to load descriptors
- `tu_shader_create`
  - `nir_lower_explicit_io` lowers `nir_intrinsic_load_deref` to
    `nir_intrinsic_load_ubo`
    - the deref, which is a `nir_deref_type_struct`, is also lowered to an
      offset by `nir_explicit_io_address_from_deref`
  - `nir_intrinsic_vulkan_resource_index` is lowered to vec3
    - x is `set`, a constant,
    - y is `offset` to the descriptor set iova (units are 64-bytes)
    - z is shift
  - `nir_intrinsic_load_vulkan_descriptor` passes the vec3 along and forces z
    to 0, because the address format is
    `nir_address_format_vec2_index_32bit_offset`
  - `lower_ssbo_ubo_intrinsic` replaces `(set, offset)` by
    `nir_intrinsic_bindless_resource_ir3(set, offset)`
- `ir3_nir_lower_variant`
  - `nir_intrinsic_bindless_resource_ir3` is untouched
  - `nir_intrinsic_load_ubo` has several outcomes
    - `ir3_nir_lower_ubo_loads` can potentially lower it to
      `nir_intrinsic_load_uniform`
      - `ir3_nir_lower_preamble` can further lower it to
      	`nir_intrinsic_copy_ubo_to_uniform_ir3`
    - otherwise, `nir_lower_ubo_vec4` lowers it to
      `nir_intrinsic_load_ubo_vec4`
- `emit_instructions`
  - `nir_intrinsic_bindless_resource_ir3` is translated to `mov`
    - remember that its src is an offset (in units of 64-bytes)
  - `nir_intrinsic_load_ubo_vec4` is translated to `ldc`
    - because it loads from a bindless resource, `ir3_handle_bindless_cat6`
      sets `IR3_INSTR_B` and sets `base` to `nir_intrinsic_desc_set`
  - `nir_intrinsic_load_uniform` is translated to `mov` from the const reg
    file
  - `nir_intrinsic_copy_ubo_to_uniform_ir3` is translated to `ldc.k`
- bind descriptor set
  - update `REG_A6XX_SP_BINDLESS_BASE` to sets' iovas for all 5 sets
  - update `REG_A6XX_HLSQ_BINDLESS_BASE` to sets' iovas for all 5 sets
  - `A6XX_HLSQ_INVALIDATE_CMD` `A6XX_HLSQ_INVALIDATE_CMD_GFX_BINDLESS`

## IR3 RA

- `ir3_calc_liveness`
  - for each block, assign `block->index`
  - for each dst in each block, assign `dst->name`
  - for each block, allocate a bitset for `live_in` and a bitset for
    `live_out`
  - `live_in` are regs used in a block before definitions
    - that is, regs that are initialized by a preceding block
  - `live_out` are regs that are used by a following block
