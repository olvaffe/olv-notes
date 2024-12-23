Mesa PanVK Compiler
===================

## Environment Variables

- `PANVK_DEBUG`
  - `nir` dumps lowered nir before passing it to the backend compiler
  - `trace` implies `sync` and dumps all submits
- `BIFROST_MESA_DEBUG` is for the bifrost compiler
  - `msgs` is useless
  - `shaders` prints shaders at different phases, including
    - final nir
    - lowered and optimized bir
    - ra'ed bir
    - final bir (except `bi_pack_valhall` makes some final touches)
    - disassembly
  - `shaderdb` prints shaderdb stats
  - `verbose` prints isa hex in addition to disassembly
  - `internal` includes internal shaders (e.g., meta, tile preload, blend) for
    dumping or shaderdb
  - `nosched` is not used on v9+
  - `nopsched` disables instr scheduling
  - `inorder` is not used on v9+
  - `novalidate` disables bir validation even on debug build
  - `noopt` disables bir optimizations
  - `noidvs` disables idvs for vs
  - `nosb` disables scoreboarding (and serializes all async instrs)
  - `nopreload` is not used on v8+
  - `spill` mimics a small reg file (r0..r7 and r56..r63) to test spilling

## `vk_device_shader_ops` and `vk_shader_ops`

- `panvk_per_arch(device_shader_ops)`
  - `panvk_get_nir_options` is called for spirv-to-nir translation or for nir
    deserialization
    - `vk_common_CreateShadersEXT` calls it
    - `vk_common_CreateGraphicsPipelines` and
      `vk_common_CreateComputePipelines` call it too
      - to translate on cache miss or to deserialize on cache hit
  - `panvk_get_spirv_options` is called for spirv-to-nir translation
    - `vk_common_CreateShadersEXT`, `vk_common_CreateGraphicsPipelines`, and
      `vk_common_CreateComputePipelines` call it
  - `panvk_preprocess_nir` is called to pre-process nir right after
    spirv-to-nir translation
    - it is called after translation and before serialization (for cache)
  - `panvk_hash_graphics_state` hashes `vk_graphics_pipeline_state` for cache
  - `panvk_compile_shaders` compiles nir to `vk_shader`
  - `panvk_deserialize_shader` deserializes blob to `vk_shader`
  - `panvk_cmd_bind_shaders` updates `cmd` shader states
    - `vk_common_CmdBindPipeline` calls `vk_graphics_pipeline_cmd_bind` which
      calls it
  - `vk_cmd_set_dynamic_graphics_state` updates `cmd->dynamic_grahpics_states`
    from a pipeline
    - `vk_common_CmdBindPipeline` calls `vk_graphics_pipeline_cmd_bind` which
      calls it
- `panvk_shader_ops`
  - `panvk_shader_destroy` destroys `vk_shader`
  - `panvk_shader_serialize` serializes `vk_shader` to blob
  - `panvk_shader_get_executable_properties`
    - `vk_common_GetPipelineExecutablePropertiesKHR` calls
      `vk_{graphics,compute}_pipeline_get_executable_properties` which calls
      it
  - `panvk_shader_get_executable_statistics`
    - `vk_common_GetPipelineExecutableStatisticsKHR` calls
      `vk_{graphics,compute}_pipeline_get_executable_statistics` which calls it
  - `panvk_shader_get_executable_internal_representations`
    - `vk_common_GetPipelineExecutableInternalRepresentationsKHR` calls
      `vk_{graphics,compute}_pipeline_get_internal_representations` which calls it
- `panvk_get_nir_options` returns `bifrost_nir_options_v9`
  - it is defined by `DEFINE_OPTIONS` macro
- `panvk_get_spirv_options`
  - `ubo_addr_format` and `ssbo_addr_format` are both
    `nir_address_format_vec2_index_32bit_offset`
  - `phys_ssbo_addr_format` is `nir_address_format_64bit_global`
- `panvk_preprocess_nir`
  - `vk_spirv_to_nir` applies generic lowering already
  - if fs, `nir_lower_io_to_vector`
  - `nir_lower_io_to_temporaries` lowers io vars to temps, and initial/final
    copies from/to the original io vars
  - `nir_lower_indirect_derefs` lowers indirect dererfs 
    - when an array is dynamically indexed, it is replaced by a binary search
      of the real index
  - `nir_opt_copy_prop_vars` copy propogates for variables
  - `nir_opt_combine_stores` combines stores to vecs
    - to update a vec component, nir loads the vec, updates a the comp, and stores back
    - after copy propation, multiple updates become a single load but multiple
      updates and multiple stores
    - this pass replaces the multiple stores by a single store
  - `nir_opt_loop`
  - if fs, `nir_lower_input_attachments`
  - `nir_lower_tex` lowers tex ops
    - e.g., apply `nir_tex_src_projector` manually
  - `nir_lower_system_values`
    - e.g., a `nir_intrinsic_load_deref` of `gl_GlobalInvocationID` is lowered
      to a `nir_load_base_global_invocation_id` a plus
      `nir_load_global_invocation_id`
  - `nir_lower_compute_system_values`
    - e.g., lower `nir_intrinsic_load_global_invocation_id` to manual
      compuatations
  - if fs, `nir_lower_wpos_center`
  - `nir_split_var_copies`
  - `nir_lower_var_copies`
- `panvk_compile_shaders`
  - `panvk_lower_nir`
    - `panvk_per_arch(nir_lower_descriptors)` lowers `vulkan_resource_index`
      and `load_vulkan_descriptor`
    - `nir_split_var_copies`
    - `nir_lower_var_copies`
    - `nir_lower_explicit_io`
      - e.g., lower a store to a varaible in ssbo storage to a
        `nir_intrinsic_store_ssbo` with an explicit addr
    - `valhall_lower_get_ssbo_size`
    - `valhall_pack_buf_idx`
    - if cs, `nir_lower_vars_to_explicit_types` and `nir_lower_explicit_io`
    - `nir_assign_io_var_locations`
    - `nir_shader_gather_info`
    - `pan_shader_preprocess`
      - `bifrost_preprocess_nir`
    - if vs, `pan_lower_image_index`
    - `panvk_lower_sysvals`
  - `panvk_compile_nir`
    - `GENX(pan_shader_compile)`
      - `bifrost_compile_shader_nir`
  - `panvk_shader_upload`
    - the binary is uploaded to `dev->mempools.exec`
    - the shader program descriptor (spd) is uploaded to `dev->mempools.rw`
    - if vs, there are actually 3 binaries and 3 spds: `pos_points`,
      `pos_triangles`, and `var`

## Resources

- a ssbo store looks like
  - `32x3  %1 = @vulkan_resource_index (%0 (0x0)) (desc_set=0, binding=0, desc_type=SSBO)`
    - `%1` refers to the descriptor at index `%0` of set 0 and binding 0
  - `32x3  %2 = @load_vulkan_descriptor (%1) (desc_type=SSBO)`
    - `%2` refers to the ssbo
  - `32x3  %3 = deref_cast (SSBO *)%2 (ssbo SSBO)  (ptr_stride=0, align_mul=0, align_offset=0)`
    - `%3` is the ssbo casted a pointer of `struct SSBO`
  - `32x3  %4 = deref_struct &%3->data (ssbo uint[])  // &((SSBO *)%2)->data`
    - `%4` refers to the `data` memeber of `struct SSBO`j which is a `uint[]`
  - `32x3  %5 = deref_array &(*%4)[%6] (ssbo uint)  // &((SSBO *)%2)->data[%6]`
    - `%5` refers to `data[%6]`
  - `@store_deref (%5, %7) (wrmask=x, access=none)`
    - stores `%7` to `%5`
- `panvk_per_arch(nir_lower_descriptors)`
  - the goal is to lower `vulkan_resource_index` and `load_vulkan_descriptor`
  - remember that
    - a hw descriptor has size `PANVK_DESCRIPTOR_SIZE`
    - a combined image sampler takes up 2 hw descriptors
    - the other descriptors take up 1 hw descriptor
  - since v8+, the addr format is
    `nir_address_format_vec2_index_32bit_offset`, a vec3
  - `collect_instr_desc_access` parses nir and calls `record_binding` on all
    used descriptors
    - `ctx->desc_info.used_set_mask` is set
    - the rest is mostly for dynamic buffers on v9+
  - `create_copy_table`
    - it is only for dynamic buffers on v9+
  - `upload_shader_desc_info`
    - `shader->desc_info.used_set_mask` is copied from `desc_info->used_set_mask`
    - the rest is mostly for dynamic buffers on v9+
  - `lower_descriptors_instr` lowers nir
    - `build_res_index` lowers `nir_intrinsic_vulkan_resource_index` to a vec3
      - the instr has a static set, a static binding, and a dynamic index
      - `shader_desc_idx` translates the set/binding to a u32
        - the higher 8 bits are for set
         - `panvk_per_arch(cmd_prepare_shader_res_table)` reserves the first set
           for driver internal set
         - as such, the set number is offset by one
        - the lower 24 bits are for hw descriptor idx in the set
      - the returned vec3 consists of
        - comp 0 is the u32 for the static set/binding
        - comp 1 is the dynamic array index
        - comp 2 is the descriptor array size minus 1
    - `build_buffer_addr_for_res_index` lowers
      `nir_intrinsic_load_vulkan_descriptor` to a vec3
      - the instr takes the res idx from `nir_intrinsic_vulkan_resource_index`
      - if robustness, it caps the dynamic array index to the descriptor array size
      - the returned vec3 consists of
        - comp 0 is propagated
        - comp 1 is propagated after optional capping
        - comp 2 is 0
- `nir_lower_explicit_io` lowers the derefs/laods/stores
  - `lower_explicit_io_deref` lowers derefs
    - `nir_deref_type_cast` is nop
    - `nir_deref_type_struct` adds the member offset to the vec3, if non-zero
      - comp 2 is the offset
    - `nir_deref_type_array` adds the element offset to the vec3
  - `lower_explicit_io_access` lowers deref loads/stores
    - `build_explicit_io_load` lowers `nir_intrinsic_load_deref`
    - `build_explicit_io_store` lowers `nir_intrinsic_store_deref`
      - if ssbo, to `nir_intrinsic_store_ssbo`
      - src 0 is the store value
      - src 1 is the first 2 components (set/binding and dynamic array index)
        of the vec3
      - src 2 is the last component (offset) of the vec3
- `valhall_lower_get_ssbo_size` lowers `nir_intrinsic_get_ssbo_size`
  - src 0 is vec2
    - comp 0 is set/binding
    - comp 1 is dynamic array index
  - for ssbo, `write_buffer_desc` writes `MALI_BUFFER` to the hw descriptor
    - dw 0 is descriptor type (buffer) and buffer type (simple)
    - dw 1 is the buffer size
    - dw 2..3 is the buffer va
  - the instr is lowered to `nir_load_ubo` to load the buffer size from the
    descriptor directly
    - `LD_BUFFER.*` treats `VALHALL_RESOURCE_TABLE_IDX` (62) specially
- `valhall_pack_buf_idx` fixes load/store addrs
  - `panvk_per_arch(nir_lower_descriptors)` uses
    `nir_address_format_vec2_index_32bit_offset`
  - the backend compiler expects `nir_address_format_32bit_index_offset`
  - this pass adds the two components of the vec2 to become u32
- `bi_emit_intrinsic` lowers nir intrinsics to bir
  - `bi_emit_load_ubo` lowers `nir_intrinsic_load_ubo` to `LD_BUFFER.i32`

## System Values

- `panvk_lower_sysvals` lowers sysvals to push consts
  - e.g., `nir_intrinsic_load_first_vertex` is lowered to
    `nir_load_push_constant` with offset 256 plus
    `offsetof(panvk_graphics_sysvals, vs.first_vertex)`
- on draw,
  - `prepare_sysvals` updates sysvals
  - `prepare_push_uniforms` uploads both push consts and sysvals
- `bi_emit_load_push_constant` lowers nir to bir
  - `BI_OPCODE_MOV_I32` or `BI_OPCODE_COLLECT_I32`
  - each src directly refers to fau

## Bifrost Compiler

- `bifrost_preprocess_nir`
  - `nir_lower_vars_to_ssa`
  - if vs,
    - `nir_lower_viewport_transform`
    - `nir_lower_point_size`
  - `nir_lower_global_vars_to_local`
  - `nir_lower_vars_to_scratch`
  - `nir_lower_indirect_derefs`
  - `nir_split_var_copies`
  - `nir_lower_var_copies`
  - `nir_lower_vars_to_ssa`
  - `nir_lower_io`
  - `nir_opt_constant_folding`
  - `nir_lower_mediump_io`
  - if vs,
    - `pan_nir_lower_store_component`
  - if fs,
    - `bi_lower_sample_mask_writes`
    - `bifrost_nir_lower_load_output`
  - `nir_lower_mem_access_bit_sizes`
  - `bi_lower_load_push_const_with_dyn_offset`
  - `nir_lower_ssbo`
  - `pan_lower_sample_pos`
  - `nir_lower_bit_size`
  - `nir_lower_64bit_phis`
  - `pan_lower_helper_invocation`
  - `nir_lower_int64`
  - `nir_opt_idiv_const`
  - `nir_lower_idiv`
  - `nir_lower_tex`
  - `nir_lower_image_atomics_to_global`
  - `nir_lower_alu_to_scalar`
  - `nir_lower_load_const_to_scalar`
  - `nir_lower_phis_to_scalar`
  - `nir_lower_flrp`
  - `nir_lower_var_copies`
  - `nir_lower_alu`
  - `nir_lower_frag_coord_to_pixel_coord`
- `bifrost_compile_shader_nir`
  - `pan_nir_lower_zs_store`
  - `bi_optimize_nir`
  - `bi_compile_variant`
    - `emit_cf_list` converts nir to bir
    - `bi_lower_swizzle`
    - optimize
      - `bi_opt_copy_prop`
      - `bi_opt_constant_fold`
      - `bi_opt_mod_prop_forward`
      - `bi_opt_mod_prop_backward`
      - `bi_opt_dce`
      - `bi_opt_cse`
    - `bi_lower_opt_instructions`
    - `va_optimize`
    - `va_lower_isel`
    - `va_lower_constants`
    - `va_repair_fau`
    - `bi_lower_branch`
    - if fs, `bi_analyze_helper_requirements`
    - optimize
      - `bi_opt_fuse_dual_texture`
      - `bi_opt_cse`
      - `bi_opt_dce`
    - `bi_pressure_schedule`
    - `bi_register_allocate`
    - `bi_opt_post_ra`
    - `va_assign_slots`
    - `va_insert_flow_control_nops`
    - `va_merge_flow`
    - `va_mark_last`
    - `bi_pack_valhall`

## Registers

- `MALI_SHADER_PROGRAM` can preload regs
  - vertex shader
    - `r59` is idvs output index
    - `r60` is `vertex_id`
    - `r61` is `instance_id`
    - `r62` is `draw_id`
  - fragment shader
    - `r58` is `!front_face`
    - `r59` is `pixel_coord`
    - `r60` is current sample mask
    - `r61` is `bary` or `sample_id` plus `sample_mask_in`
    - `r62` is `layer_id`
  - compute shader
    - `r55`..`r56` are `local_invocation_id`
    - `r57`..`r59` are `workgroup_id`
    - `r60`..`r62` are `global_invocation_id`
  - blend shader is special
    - `r0`..`r3` are `src_color`
    - `r4`..`r7` are `src_color2`, for dual-source blending
    - `r48` is the return addr of fs

## BIR

- `bi_index` is a pseudo reg
  - `BI_INDEX_NULL` is an unused reg
    - `value` is 0
  - `BI_INDEX_NORMAL` is an ssa reg
    - `value` is the ssa id
  - `BI_INDEX_REGISTER` is a physical reg
    - `value` is the reg id
  - `BI_INDEX_CONSTANT` is a 32-bit imm
    - `value` is the imm
  - `BI_INDEX_PASS` is a passthrough and is only before v9
    - `value` is `enum bifrost_packed_src`
  - `BI_INDEX_FAU` is a fau val
    - `value` is `enum bir_fau`
      - `BIR_FAU_ZERO` is unused
      - `BIR_FAU_LANE_ID` is `VA_FAU_SPECIAL_PAGE_3_LANE_ID`
      - `BIR_FAU_WARP_ID` is unused
      - `BIR_FAU_CORE_ID` is unused
      - `BIR_FAU_FB_EXTENT` is unused
      - `BIR_FAU_ATEST_PARAM` is `VA_FAU_SPECIAL_PAGE_0_ATEST_DATUM`
      - `BIR_FAU_SAMPLE_POS_ARRAY` is `VA_FAU_SPECIAL_PAGE_0_SAMPLE`
      - `BIR_FAU_BLEND_0` is `VA_FAU_SPECIAL_PAGE_0_BLEND_DESCRIPTOR_0`
      - `BIR_FAU_TYPE_MASK` is unused
      - `BIR_FAU_TLS_PTR` is `VA_FAU_SPECIAL_PAGE_1_THREAD_LOCAL_POINTER`
      - `BIR_FAU_WLS_PTR` is `VA_FAU_SPECIAL_PAGE_1_WORKGROUP_LOCAL_POINTER`
      - `BIR_FAU_PROGRAM_COUNTER` is `VA_FAU_SPECIAL_PAGE_3_PROGRAM_COUNTER`
      - `BIR_FAU_UNIFORM` means the lower 5 bits are an idx into fau
        - the fau is 512 bytes in size
        - it is indexed as dwords and indices are thus 7 bits
        - the higher 2 bits are fau page and are encoded in bit 57 and 58 of
          the instr
      - `BIR_FAU_IMMEDIATE` means the lower 5 bits are an idx into an imm
        table for common immediate values
        - see `ISA.xml`
- `bi_instr` is a pseudo instr

## `ISA.xml` and `valhall.py`

- an instruction has 64 bits
  - see `va_pack_instr` for definitive definitions
  - bit 0..7: src0
  - bit 8..15: src1
  - bit 16..23: src2
  - bit 24..39: modifiers
  - bit 40..47: dst
  - bit 48..56: primary opcode
  - bit 57..58: fau page
    - an src reg may refer to fau for a preloaded value
    - different preloaded values may be on different 128-byte fau pages
      specified by this field
  - bit 59..62: flow
    - `VA_FLOW_NONE` is the default
    - `VA_FLOW_WAIT*` waits for different slots
    - `VA_FLOW_RECONVERGE` is set when exec mask changes
    - `VA_FLOW_DISCARD` terminates an fs helper invocation
      - for explicit `dFdx`/`dFdy` or implicit `texture` to work in fs, helper
        invocations are created
      - this terminates them when they are no longer needed
    - `VA_FLOW_END` terminates a thread
  - bit 63: reserved
- `build_instr` parses a `<ins>` to an `Instruction`
  - `build_source` parses each `<src>` to a `Source` in order
    - pseudo srcs are ignored
    - real srcs take up bit 0..5, 8..13, or 16..21 in order
  - parses each `<dest>` to a `Dest`
    - there is no `<dest>` so this is unused
  - parses `srcs=` to `Source`
    - there is no `srcs=` so this is unsued
  - parses `dests=` to `Dest`
    - there is no `srcs=` so this is unsued
  - `build_staging` parses each `<sr>` to a `Staging` in order
    - if write and no flags, it takes up bit 16..21
    - otherwise, it takes up bit 40..45
      - flags takes up bit 46..47
  - `build_imm` parses each `<imm>` to an `Immediate` in order
  - `build_modifier` parses each `<va_mod>` to a `Modifier` in order
  - maps remaining xml tags to `Modifier`s
    - it maps using `MODIFIERS`
    - pseudo modifiers are ignored
- `TEX_FETCH`
  - bit 0..5: `<src size="64">Image to read from</src>`
  - (bit 6..7: src flags)
  - bit 8: `<wide_indices/>`
  - bit 9: `<array_enable/>`
  - bit 10: `<texel_offset/>`
  - (bit 11..15: unused)
  - bit 16..21: `<sr write="true" flags="false"/>`
    - base staging reg to write
    - they are for texels
  - bit 22..25: `<write_mask/>`
  - bit 26..27: `<register_type/>`
    - it is supposed to be float, signed, or unsigned, but is never set
  - bit 28..29: `<dimension/>`
  - bit 30..32: `<slot/>`
    - a message is async and a slot is specified
    - the flow field can be used to wait for a slot
  - bit 33..35: `<sr_count/>`
    - number of staging regs to read
    - they are for coords and depend on dim
  - bit 36..38: `<sr_write_count/>`
    - number of staging regs to write minus 1
    - typically `4-1=3` (f32) or `2-1=1` (f16)
  - bit 39: `<skip/>`
  - bit 40..45: `<sr read="true" flags="false"/>`
    - base staging reg to read
    - they are for coords
  - bit 46: `<register_width/>`
    - 32 when set; 16 when clear
  - (bit 47: unused)
  - bit 48..56: primary opcode
  - bit 57..58: fau page
  - bit 59..62: flow
  - bit 63: reserved

## ISA

- `LD_BUFFER.i32`
  - src 0 is byte offset
  - src 1 is mode descriptor
    - the cs sets up SRT which points to an array of `MALI_RESOURCE`
      - each `MALI_RESOURCE` points to a descriptor set
      - each descriptor set has an array of 32-byte descriptors
    - when the highest byte is less than 16(?),
      - the highest byte is the index of `MALI_RESOURCE` array
      - the lower bytes are the index of the descriptor array
      - the instr loads from the buffer referenced by the descriptor
    - when the highest byte is 62(?),
      - the lower bytes are the index of `MALI_RESOURCE` array
      - the instr loads from the descriptor array
- `LOAD.i*` and `STORE.i*`
  - `va_pack_memory_access` packs `bi_seg` to `va_memory_access`
    - `BI_SEG_TL` (thread-local) to `VA_MEMORY_ACCESS_FORCE`
    - `BI_SEG_POS` (pos sh output) to `VA_MEMORY_ACCESS_ISTREAM`
    - `BI_SEG_VARY` (vary sh output) to `VA_MEMORY_ACCESS_ESTREAM`
    - the rest are to `VA_MEMORY_ACCESS_NONE` (no hint)
- `TEX_FETCH`
  - `bi_emit_tex_valhall` and `bi_tex_fetch_to`
    - `I->dest[0]` is `dest` which is a temp for fetched texels
    - `I->src[0]` is `idx` which is the texcoords
    - `I->src[1]` is `src0` which is the idx for sampler desc
    - `I->src[2]` is `src1` which is the idx for texture desc
    - descriptor index packing
      - the descriptor indices for sampler and texture are formed by
        `pan_res_handle`
      - when `va_is_valid_const_narrow_index` is true for both, they can be
        packed to u32
        - bit 0..10: sampler index
        - bit 11..15: sampler table
        - bit 16..26: texture index
        - bit 27..31: texture table
  - `va_pack_instr` packs
    - bit 0..7: `va_pack_src(I, 1)`
    - bit 16..21: `va_pack_reg(I, I->dest[0])`
    - bit 33..35: `bi_count_read_registers(I, 0)`
    - bit 36..38: `bi_count_write_registers(I, 0) - 1`
    - bit 40..45: `va_pack_reg(I, I->src[0])`

## VS

- `prepare_vs_driver_set` preps the driver-internal descriptor set
  - `MALI_ATTRIBUTE` descriptors
    - there are `MAX_VS_ATTRIBS` (16) of them
    - only used attrs are initialized
    - `type` is `MALI_DESCRIPTOR_TYPE_ATTRIBUTE`
    - `attribute_type` is usually `MALI_ATTRIBUTE_TYPE_1D` unless instanced
    - `offet_enable` is true unless instanced
    - `format` is from `GENX(panfrost_format_from_pipe_format)`
    - `table` is always 0
    - `frequency` is `MALI_ATTRIBUTE_FREQUENCY_VERTEX` unless instanced
    - `divisor_r`, `divisor_d`, `divisor_e` are for instancing
    - `offset` is the offset of this attr in the vertex data
    - `buffer_index` is the index of the buffer descriptor
    - `stride` is the size of the vertex data
    - `packet_stride` and `attribute_stride` are probably for the unused
      `MALI_ATTRIBUTE_TYPE_VERTEX_PACKET`
  - `MALI_SAMPLER` descriptor
    - this is dummy
  - `MALI_BUFFER` descriptors
    - one for each vb
    - `address` and `size` are va and size of the vb
- suppose we have a vs that
  - `gl_Position = in_attr;`
  - `out_texcoord = in_attr;`
- `panvk_lower_nir`
  - it assigns driver locations for in-vars as offsets to
    `VERT_ATTRIB_GENERIC0`
  - `nir_assign_io_var_locations` assigns driver locations for out-vars
    sequentially
  - `pan_shader_preprocess` calls `bifrost_preprocess_nir`
    - `nir_lower_viewport_transform`
      - `gl_Position` is in clip coordinates
      - hw expects framebuffer coordinates
      - this pass applies viewport manually
        - `nir_load_viewport_offset` and `nir_load_viewport_scale` load the
          viepwort
    - `nir_lower_io` lowers vs io
      - `lower_load` lowers `nir_intrinsic_load_deref` of in-vars to
        `nir_intrinsic_load_input`
        - `base` is `var->data.driver_location`
          - this is `var->data.location - VERT_ATTRIB_GENERIC0`
        - `component` is based on `var->data.location_frac` which is 0 unless
          in-vars are packed
        - `io_semantics.location` is `var->data.location`
          - this is `VERT_ATTRIB_GENERIC0` or later
        - `src` is the offset, which is 0 unless in-vars are packed
      - `lower_store` lowers `nir_intrinsic_store_deref` of out-vars to
        `nir_intrinsic_store_output`
        - `base` is `var->data.driver_location`
          - this is assigned sequentially by `nir_assign_io_var_locations`
        - `component` is based on `var->data.location_frac`, which is 0 unless
          out-vars are packed
        - `io_semantics.location` is `var->data.location`
          - `VARYING_SLOT_POS` for `gl_Position`
          - `VARYING_SLOT_VAR0` or later for generic varyings
        - `src0` is the value
        - `src1` is the offset, which is 0 unless out-vars are packed
  - `panvk_lower_sysvals`
    - various sysvals are lowered to `nir_load_push_constant`
      - including `nir_load_viewport_offset` and `nir_load_viewport_scale`
- `bifrost_compile_shader_nir`
  - `info->vs.idvs` is set to `bi_should_idvs`
    - it is always true for vs on panvk
  - `pan_nir_collect_varyings`
  - `bi_compile_variant` with `BI_IDVS_POSITION`
    - `ctx->malloc_idvs` is always true on panvk
    - `bifrost_nir_specialize_idvs` eliminates all outs but
      - `VARYING_SLOT_POS`
      - `VARYING_SLOT_PSIZ`
      - `VARYING_SLOT_LAYER`
  - `bi_compile_variant` with `BI_IDVS_VARYING`
    - `ctx->malloc_idvs` is always true on panvk
    - `bifrost_nir_specialize_idvs` keeps all outs but
      - `VARYING_SLOT_POS`
      - `VARYING_SLOT_PSIZ`
      - `VARYING_SLOT_LAYER`
- `bi_emit_load_attr` translates `nir_intrinsic_load_input`
  - `bi_is_imm_desc_handle`
    - remember that that the first `MAX_VS_ATTRIBS` descriptors of set 0 is
      `MALI_ATTRIBUTE`
    - `table_index` is always 0
    - `res_index` is `base` plus `offset`
    - returns true
  - `bi_ld_attr_imm_to` emits `BI_OPCODE_LD_ATTR_IMM`
    - src0 is vertex id
    - src1 is instance id
    - `attribute_index` is the index of the `MALI_ATTRIBUTE` descriptor
    - this fetches the attr for the vertex/instance and converts the format
  - `va_res_fold_table_idx` sets table, which is always 0
- `bi_emit_store_vary` translates `nir_intrinsic_store_output`
  - we need to query the addr to store the out-var to
    - the regular path is to do a `BI_OPCODE_LEA_ATTR_IMM`
      - the hw has generated descriptors for all out-vars
      - this op takes a descriptor index, vertex id, and instance id
      - it returns the addr to write the out-var to
    - the fast path is to do a `BI_OPCODE_LEA_BUF_IMM` on preloaded `r59`
  - `bi_lea_buf_imm` emits `BI_OPCODE_LEA_BUF_IMM` to get the 64-bit va
  - `bi_emit_split_i32` emits pseudo `BI_OPCODE_SPLIT_I32`
    - the lea loads a 64-bit va to an ssa reg
    - this pseudo instr splits it into two 32-bit ssa regs
    - i guess this tells ra to use two consecutive phy regs to hold the
      value
  - `bi_store` emits `BI_OPCODE_STORE_I*`
    - for vec4, it uses `BI_OPCODE_STORE_I128`
    - src0 is the value
    - src1 and src2 are the va
    - `seg` is the special `BI_SEG_POS` or `BI_SEG_VARY`
    - `byte_offset` is 0 or `bi_varying_offset()`
      - note that panvk does not set `fixed_varying_mask` and
        `bi_varying_offset` returns `16 * (sem.location - VARYING_SLOT_VAR0)`
      - the size of `BI_SEG_VARY` is specified by cs `r48`

## FS

- `prepare_fs_driver_set` preps the driver-internal descriptor set
  - `MALI_SAMPLER` descriptor
    - this is dummy
- suppose we have a fs that
  - `out_color = in_color;`
- `panvk_lower_nir`
  - `nir_assign_io_var_locations` assigns driver locations for in-vars and
    out-vars sequentially
  - `pan_shader_preprocess` calls `bifrost_preprocess_nir`
    - `nir_lower_io` lowers fs io
      - `lower_load` lowers `nir_intrinsic_load_deref` of in-vars to
        `nir_intrinsic_load_barycentric_pixel` and
        `nir_intrinsic_load_interpolated_input`
        - similar to vs
        - `io_semantics.location` is `var->data.location`
          - this is `VARYING_SLOT_VAR0` or later
      - `lower_store` lowers `nir_intrinsic_store_deref` of out-vars to
        `nir_intrinsic_store_output`
        - similar to vs
        - `io_semantics.location` is `var->data.location`
          - this is `FRAG_RESULT_DATA0` or later
  - `panvk_lower_sysvals`
- `bifrost_compile_shader_nir`
  - `bi_compile_variant` with `BI_IDVS_NONE`
    - `ctx->malloc_idvs` is always true on panvk
- `bi_emit_load_vary` translates `nir_intrinsic_load_barycentric_pixel` and
  `nir_intrinsic_load_interpolated_input`
  - `bi_varying_src0_for_barycentric` returns preloaded `r61`
  - `bi_ld_var_buf_imm_to` emits `BI_OPCODE_LD_VAR_BUF_IMM_F32`
    - src0 is barycentric params
    - `register_format` is `BI_REGISTER_FORMAT_F32` for 32-bit components
    - `sample` is `BI_SAMPLE_CENTER`
    - `source_format` is `BI_SOURCE_FORMAT_F32`
    - `update` is `BI_UPDATE_STORE`
    - `vecsize` depends on component count
    - `index` is the varying offset in `BI_SEG_VARY`
- `bi_emit_fragment_out` translates `nir_intrinsic_store_output`
  - `bi_emit_atest` emits `BI_OPCODE_ATEST`
    - src0 is preloaded r60, the current sample mask
    - src1 is the alpha component of the color output
    - src2 is `BIR_FAU_ATEST_PARAM`
    - this performs alpha-to-coverage and updates r60
      - `bi_allocate_registers` allocs r60 for dest
  - `bi_emit_blend_op` emits `BI_OPCODE_BLEND`
    - src0 is the color (4 consecutive regs)
    - src1 is the r60, the current sample mask
    - src2 and src3 are 64-bit `BIR_FAU_BLEND_0`
      - this is the `MALI_INTERNAL_BLEND` descriptor
    - src4 is for dual color
    - `bi_allocate_registers` allocs r48 for dest, r0..r3 for src0, and r4..r7
      for src4
      - blend shader assumes those regs are preloaded
- `bi_pack_valhall` calls `va_lower_blend`
  - it finds `BI_OPCODE_BLEND` and appends instrs it
  - `BI_OPCODE_BLEND` checks the blend descriptor
    - if fixed-function blending, it sends the color to the blender and
      terminates fs
    - if blend shader, it is nop and continues to instrs appended here
  - `bi_iadd_imm_i32_to` sets r48 to 0
    - if r48 is non-zero, blend shader will branch back to the addr
  - `bi_branchzi` branches to the blend shader

## Blend Shader

- when `BI_OPCODE_BLEND` executes in fs, it either sends the color to the
  fixed-function blender or is nop
  - fs is supposed to follow the nop with a branch to the blend shader
- `prepare_blend` calls `panvk_per_arch(blend_emit_descs)` to emit blend
  descriptors
  - if the blender cannot support the blend op or the logic op,
    `blend_needs_shader` returns true and `get_blend_shader` compiles a blend
    shader
  - `GENX(pan_blend_create_shader)`
    - `nir_load_interpolated_input` loads the color inputs
      - `VARYING_SLOT_COL0` for color
      - `VARYING_SLOT_VAR0` for dual source color
    - `nir_store_output` stores the color outputs
    - `nir_lower_blend` simulates blending in nir
      - `nir_load_output` fetches the fb outputs
  - `lower_load_blend_const` lowers
    `nir_intrinsic_load_blend_const_color_rgba` to `nir_load_push_constant`
  - `panfrost_compile_inputs`
    - `is_blend` is true
    - `blend.bifrost_blend_desc` is an embedded 64-bit `MALI_INTERNAL_BLEND`
  - `pan_shader_preprocess` is the standard compiler prepass
    - `bifrost_nir_lower_load_output` lowers `nir_intrinsic_load_output` (fb
      fetch) to 
      - `nir_load_rt_conversion_pan`
      - `nir_load_converted_output_pan`
  - `GENX(pan_inline_rt_conversion)` lowers
    `nir_intrinsic_load_rt_conversion_pan` to an imm for dw1 of
    `MALI_INTERNAL_BLEND`
- `bi_emit_load_blend_input` translates
  `nir_intrinsic_load_interpolated_input` to input color
  - it uses r0..r3 or r4..7 as set up by the fs
- `bi_emit_ld_tile` translates `nir_intrinsic_load_converted_output_pan` to
  `BI_OPCODE_LD_TILE`
  - remember that fb fetch just fetches from tilebuffer
  - src0 is `bi_pixel_indices`, the pixel to fetch
  - src1 is `bi_coverage`, the current sample mask
  - src2 is conversion descriptor
    - it is an imm from lowering of `nir_intrinsic_load_rt_conversion_pan` in
      `GENX(pan_inline_rt_conversion)`
- `bi_emit_fragment_out` translates `nir_intrinsic_store_output`
  - remember that `is_blend` is set
  - atest is skipped
  - `bi_emit_blend_op` emits `BI_OPCODE_BLEND` using
    `inputs->blend.bifrost_blend_desc` rather than `BIR_FAU_BLEND_0`
  - `bi_branchzi` either branches back to fs or terminates

## Frame Shader

- `RUN_FRAGMENT` uses `MALI_FRAMEBUFFER`
  - 64-byte `MALI_FRAMEBUFFER_PARAMETERS`
  - 64-byte `MALI_FRAMEBUFFER_PADDING`
  - multiple 64-byte `MALI_RENDER_TARGET` for RTs
- `MALI_FRAMEBUFFER_PARAMETERS` has pre/post frame shaders
  - `frame_shader_dcds` is an array of 3 `MALI_DRAW` to define 3 frame shaders
  - `pre_frame_0`, `pre_frame_1`, and `post_frame` control which ones of them
    are enabled
    - `MALI_PRE_POST_FRAME_SHADER_MODE_NEVER`
    - `MALI_PRE_POST_FRAME_SHADER_MODE_ALWAYS`
    - `MALI_PRE_POST_FRAME_SHADER_MODE_INTERSECT`
    - `MALI_PRE_POST_FRAME_SHADER_MODE_EARLY_ZS_ALWAYS`
  - `panvk_per_arch(cmd_fb_preload)` uses `pre_frame_0` for rt preload and
    `pre_frame_1` for zs preload
- each `MALI_DRAW` has
  - `MALI_DCD_FLAGS_0`
  - `MALI_DCD_FLAGS_1`
  - `MALI_DCD_FLAGS_2`
  - va of `MALI_DEPTH_STENCIL`
  - va of `MALI_BLEND` array
  - va of `MALI_SHADER_ENVIRONMENT` which has
    - `shader` points to `MALI_SHADER_PROGRAM`
    - `resources` points to `MALI_RESOURCE`
    - `thread_storage` points to `MALI_LOCAL_STORAGE`
    - `fau` points to fau
  - IOW, it specifies an entire post-vertex pipeline
- `issue_fragment_jobs` calls `panvk_per_arch(cmd_fb_preload)` which calls
  `cmd_emit_dcd` for rt preload and zs preload
  - `get_preload_shader` compiles the frame shader
    - `get_preload_nir_shader`
      - `nir_load_pixel_coord` for coords
      - `nir_load_layer_id` for layer id if layered
      - `nir_texop_txf` or `nir_texop_txf_ms` to fetch from image
      - `nir_store_output` to store to tilebuffer
    - `panfrost_compile_inputs`
      - `is_blit` is true
  - `fill_textures` fills in `MALI_TEXTURE` array, one for each rt
  - `fill_bds` fills in `MALI_BLEND` array, one for each rt
  - `MALI_DRAW` is initialized
- frame shaders work the same way as a regular fs, except atest can
  potentially be skipped

## Texture

- `bi_emit_tex_valhall` translates a `nir_tex_instr`
  - `bi_tex_single_to` lowers `nir_texop_tex`, `nir_texop_txl`, and
    `nir_texop_txb` to `BI_OPCODE_TEX_SINGLE`
  - `bi_tex_fetch_to` lowers `nir_texop_txf` and `nir_texop_txf_ms` to
    `BI_OPCODE_TEX_FETCH`
  - `bi_tex_gather_to` lowers `nir_texop_tg4` to `BI_OPCODE_TEX_GATHER`
  - `bi_tex_gradient_to` lowers `nir_texop_lod` to `BI_OPCODE_TEX_GRADIENT`
