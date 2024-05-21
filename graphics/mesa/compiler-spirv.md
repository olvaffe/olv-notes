Mesa SPIR-V
===========

## `spirv_to_nir`

- spirv has strict requirements on the logical layout of spirv
  - all `OpCapability`
  - all `OpExtension`
  - all `OpExtInstImport`
  - one `OpMemoryModel`
  - all `OpEntryPoint`
  - all `OpExecutionMode` or `OpExecutionModeId`
  - all debug instructions
    - all `OpString` and `OpSource*`
    - all `OpName` and `OpMemberName`
    - all `OpModuleProcessed`
  - all annotation instructions
    - all `Op*Decorate*`
  - and
    - all `OpType*`
    - all `OpConstant*` and `OpSpec*`
    - all `OpVariable` whose storage class is not `Function`
    - preferred location for `OpUndef`
    - `OpLine` and `OpNoLine`, and `OpExtInst` are also allowed
  - first `OpFunction`
- `vtn_create_builder` creates the builder for an entry point
  - it parses the first 5 dwords of spirv binary
    - dword 0 is `SpvMagicNumber`
    - dword 1 is version
    - dword 2 is generator
    - dword 3 is the largest id plus 1
    - dword 4 must be zero
  - `b->values` is an array of `vtn_value`
  - `b->vars_used_indirectly` is for spirv 1.3 and lower
    - since 1.4, `OpEntryPoint` lists all global variables and does not need
      tracking
- `vtn_handle_preamble_instruction` handles the instructions up to the
  annotation instructions
- `vtn_handle_execution_mode` handles execution modes after
  `vtn_handle_preamble_instruction` parses them into decorations
- `vtn_handle_variable_or_type_instruction` handles variables and types
- `vtn_handle_execution_mode_id` handles execution mode ids
- `vtn_set_instruction_result_type` updates result types
- `vtn_build_cfg` builds the cfg
- `vtn_function_emit` emits functions
- `nir_lower_goto_ifs` lowers gotos
- `nir_lower_continue_constructs` lowers continues
- if 1.3 or lower, `nir_remove_dead_variables` removes dead variables using
  `b->vars_used_indirectly`
- `nir_opt_dce` performs dce
- reflections

## `spirv_to_nir_options`

- `enum nir_spirv_execution_environment environment` is vulkan, opencl, or
  opengl
- `bool view_index_is_input` uses `VARYING_SLOT_VIEW_INDEX` instead of
  `SYSTEM_VALUE_VIEW_INDEX`
- `bool create_library` creates a library instead
- `uint32_t float_controls_execution_mode` is the default fp mode
- `enum gl_subgroup_size subgroup_size` defines `gl_SubgroupSize`
- `bool mediump_16bit_alu` respects `SpvDecorationRelaxedPrecision`
- `bool mediump_16bit_derivatives`
- `bool amd_gcn_shader` supports `SPV_AMD_gcn_shader`
- `bool amd_shader_ballot` supports `SPV_AMD_shader_ballot`
- `bool amd_trinary_minmax` supports `SPV_AMD_shader_trinary_minmax`
- `bool amd_shader_explicit_vertex_parameter` supports
  `SPV_AMD_shader_explicit_vertex_parameter`
- `bool printf` supports `OpenCLstd_Printf`
- `const struct spirv_capabilities *capabilities` is the supported caps
- `nir_address_format ubo_addr_format` is the address format for ubo
- `nir_address_format ssbo_addr_format`
- `nir_address_format phys_ssbo_addr_format`
- `nir_address_format push_const_addr_format`
- `nir_address_format shared_addr_format`
- `nir_address_format task_payload_addr_format`
- `nir_address_format global_addr_format`
- `nir_address_format temp_addr_format`
- `nir_address_format constant_addr_format`
- `uint32_t min_ubo_alignment` should match `minUniformBufferOffsetAlignment`
- `uint32_t min_ssbo_alignment` should match `minStorageBufferOffsetAlignment`
- `const nir_shader *clc_shader` is from libclc
- `debug.func` is for logging
- `bool force_tex_non_uniform` assumes `SpvDecorationNonUniform` for tex as a
  wa
- `bool force_ssbo_non_uniform` assumes `SpvDecorationNonUniform` for ssbo as
  a wa
- `bool skip_os_break_in_debug_build` disables abort on compile failures
- `uint32_t shader_index` is for `VK_AMDX_shader_enqueue`

## `vtn_handle_preamble_instruction`

- a `vtn_value` corresponds to an `<id>`
  - `b->values` has been preallocated for all `<id>`s
- `vtn_handle_preamble_instruction` is a quick first pass to parse up to
  annotation instructions
- `SpvOpCapability` is parsed to `b->enabled_capabilities`
- `SpvOpExtInstImport` is parsed to `val->ext_handler`
- `SpvOpMemoryModel` is parsed to `b->physical_ptrs` and `b->mem_model`
- `OpEntryPoint` is parsed to
  - `val->name`
  - `b->entry_point`
  - `b->interface_ids` for used global variables
- `SpvOpName` is parsed to `val->name`
- `Spv*Decorate*` is parsed to `val->decoration`
  - note that decoration group has been deprecated
- `SpvOpExecutionMode` and `SpvOpMemberName` are parsed to `val->decoration`
  as well

## `vtn_handle_execution_mode`

- `vtn_handle_preamble_instruction` has parsed execution modes as decorations
  of the entry point
- `vtn_handle_execution_mode` parses them into `b->shader->info` (mostly) or
  `b->exact` (`SpvExecutionModeContractionOff`)
- `b->shader->info.float_controls_execution_mode` is the fp execution modes

## `vtn_handle_variable_or_type_instruction`

- `vtn_set_instruction_result_type` updates `val->type`, if the instruction
  has both and `Result <id>` and `Result Type`
- `vtn_handle_type` parses `SpvOpType*` to `val->type`
- `vtn_handle_constant` parses `SpvOpConstant*` and `SpvOpSpecConstant*` to
  `val->constant`
- `vtn_handle_variables` parses `SpvOpVariable`, `SpvOpUndef`, and
  `SpvOpConstantSampler`
  - `vtn_create_variable` creates a `vtn_pointer` and a `vtn_variable`
    - `val->pointer` points to the pointer
    - `val->pointer->var` points to the variable
    - `var->var` points to a `nir_variable`
      - it is also added to `nir_shader_add_variable`

## `vtn_build_cfg`

- `vtn_cfg_handle_prepass_instruction`
  - `SpvOpFunction` is parsed to `b->func`
    - `b->func->nir_func` is `nir_function`
  - `SpvOpFunction` is parsed to `b->func->end`
  - `SpvOpFunctionParameter` is parsed to `val->ssa`
  - `SpvOpLabel` is parsed to `b->block` and `val->block`
    - if this is the first block of a function, also to `b->func->start_block`
  - `SpvOp*Merge` is parsed to `b->block->merge`
  - `SpvOpBranch` and others are parsed to `b->block->branch`
- `vtn_build_structured_cfg`

## `vtn_function_emit`

- `vtn_emit_cf_func_structured` builds nir blocks and calls `vtn_emit_block`
  on each block
- `vtn_handle_phis_first_pass` handles `SpvOpPhi`
- `vtn_handle_body_instruction` handles non-phi instructions
  - `vtn_handle_extension`
  - `vtn_handle_variables`
  - `vtn_handle_texture`
  - `vtn_handle_image`
  - `vtn_handle_atomics`
  - `vtn_handle_select`
  - `vtn_handle_alu`
  - `vtn_handle_integer_dot`
  - `vtn_handle_bitcast`
  - `vtn_handle_composite`
  - `vtn_handle_barrier`
  - `vtn_handle_subgroup`
  - `vtn_handle_ptr`
  - `nir_begin_invocation_interlock`
  - `nir_end_invocation_interlock`
  - more
