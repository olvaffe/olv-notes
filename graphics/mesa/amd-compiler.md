Mesa RADV ACO
=============

## Overview

- `aco_compile_shader` compiles a shader
  - `aco::init` initializes `debug_flags` from `ACO_DEBUG`
  - `aco::select_program` translates nir to aco ir
    - `visit_cf_list` visits each nir control flow node to translate nir to
      aco ir instruction-by-instruction
  - `aco_postprocess_shader` schedules aco ir, performs reg alloc, optimizes,
    eliminates pseudo ops, etc.
    - `aco::lower_phis` lowers phis
    - `aco::value_numbering` performs value numbering
    - `aco::optimize` performs pre-RA optimizations
    - `aco::insert_exec_mask` inserts exec mask
    - `aco::spill` spills if necessary
    - `aco::schedule_program` schedules instructions
    - `aco::register_allocation` allocates regs
    - `aco::optimize_postRA` performs post-RA optimizations
    - `aco::ssa_elimination` eliminates ssa
    - `aco::lower_to_hw_instr` translates aco ir to hw instructions
    - `aco::schedule_vopd` schedules to use VOPD (dual-issue,
      rdna3+)
    - `aco::schedule_ilp` schedules to improve instruction-level parallelism
    - `aco::insert_wait_states` inserts `s_waitcnt`
    - `aco::insert_NOPs` inserts `s_nop`
    - `aco::form_hard_clauses` inserts `s_clause`
  - `aco::emit_program` emits the binary
  - `radv_aco_build_shader_binary` allocs a `radv_shader_binary_legacy` to
    hold the binary

## ACO IR

- `class Program`
  - `blocks` is `std::vector<Block>`
  - `temp_rc` is `std::vector<RegClass>`
  - `stage` is `Stage`
  - `create_and_insert_block` creates and adds a new block to `blocks`
  - `allocateTmp` creates and adds a new `Temp` to `temp_rc`
    - `bld.tmp` and `bld.def` make use of this method
  - `peekAllocationId` returns the number of temps
    - it returns the next alloc id, and because we allocate temps linearly, it
      is the same as temp count
- `struct Block`
  - `instructions` is `std::vector<aco_ptr<Instruction>>`
- `struct Instruction`
  - `opcode` is `aco_opcode`
  - `operands` is `aco::span<Operand>`
  - `definitions` is `aco::span<Definition>`
  - subclasses
    - `SALU_instruction`
    - `SMEM_instruction`
    - `VALU_instruction`
      - `VINTERP_inreg_instruction`
      - `VOPD_instruction`
      - `DPP16_instruction`
      - `DPP8_instruction`
      - `SDWA_instruction`
    - `VINTRP_instruction`
    - `DS_instruction`
    - `LDSDIR_instruction`
    - `MUBUF_instruction`
    - `MTBUF_instruction`
    - `MIMG_instruction`
    - `FLAT_instruction`
    - `Export_instruction`
    - `Pseudo_instruction`
      - `Pseudo_branch_instruction`
      - `Pseudo_barrier_instruction`
      - `Pseudo_reduction_instruction`
- `class Definition`
  - `temp` is `Temp`
  - `reg_` is `PhysReg`
- `class Operand`
  - `data_` is a union of `Temp`, `uint32_t`, or `float`
  - `reg_` is `PhysReg`
- `struct PhysReg`
  - `reg_b` is `uint16_t`
- `struct Temp`
  - `id_` is `uint32_t:24`
  - `reg_class` is `uint32_t:8`
- `struct RegClass`
  - `rc` is `RC`
  - `enum RC : uint8_t`
    - there are `s1`, `s2`, `s3`, `s4`, `s6`, `s8`, and `s16`
      - these represent N consecutive sgprs
    - there are `v1`, `v2`, `v3`, `v4`, `v5`, `v6`, `v7`, and `v8`
      - these represent N consecutive vgprs
    - there are `v1b`, `v2b`, `v3b`, `v4b`, `v6b`, and `v8b`
      - these represent N consecutive byte-sized pesudo vgprs
    - there are `v1_linear` and `v2_linear`
      - these represent N consecutive linear pesudo vgprs
    - bits
      - bit 0..4: N consecutive regs, up to 31
      - bit 5: vgpr rather than sgpr
      - bit 6: byte-sized
      - bit 7: linear

## `aco::select_program`

- `aco::setup_isel_context` sets up the instruction selection context
  - `aco::init_program` initializes the program
    - `sgpr_alloc_granule` and `vgpr_alloc_granule` are set here
      - llvm uses `wave64_vgpr_alloc_granularity` which is different?
    - `vgpr_limit`, max number of vgprs available to a shader, is 256 on gfx10+
  - `aco::setup_nir` calls
    - `nir_convert_to_lcssa`
    - `nir_lower_phis_to_scalar`
    - `nir_index_ssa_defs`
- `aco::init_context` initializes isel context
  - `nir_divergence_analysis` marks non-uniform ssas as `divergent`
  - it loops over all nir instructions to intialize a subrange of
    `ctx->program->temp_rc.data`
    - the goal is to assign a RC for each ssa
      - after the loop, given a `nir_def`,
        - `ctx->first_temp_id + def->index` is the temp id of the ssa
        - `ctx->program->temp_rc[id]` is the RC of the ssa
        returns its RC
    - for `nir_instr_type_alu`, there are a few possiblities
      - for moves such as `nir_op_mov`, def is vgpr or sgpr depending on
        whether the instruction is divergent or uniform
      - for float ops such as `nir_op_fmul`, def is vgpr
        - because only valu supports float ops
      - for int ops such as `nir_op_imul`, def is vgpr if it has 2 components
        - because only valu supports packed math
        - `nir_lower_alu_width` has lower the widths to 1 (mostly) or 2
          (16-bit components)
      - if any of the src is vgpr, def is vgpr
    - `get_reg_class` returns the reg class
      - `type` is vgpr or sgpr
      - `components` is number of components
      - `bitsize` is the component size
      - it returns `sX`, `vX`, or `vXb`
- `aco::visit_cf_list` visits the shader entrypoint to select instructions
  - `aco::visit_block`
    - `aco::visit_alu_instr` selects the aco instruction for a `nir_alu_instr`
    - `aco::visit_load_const`
    - `aco::visit_intrinsic`
    - `aco::visit_tex`
    - `aco::visit_phi`
    - `aco::visit_undef`
    - `aco::visit_jump`
  - `aco::visit_if`
  - `aco::visit_loop`
- instruction selection for `nir_op_mov` as an example
  - the `nir_alu_instr` has an src and a def
  - `get_ssa_temp` returns a `Temp` for the def
    - `aco::init_context` has assigned unique `(id, rc)` for each def
  - `get_alu_src` returns a `Temp` for the src
    - `get_ssa_temp` returns a `Temp` for the src
    - if src is scalar, the temp is returned
    - otherwise, `emit_extract_vector` extracts a scalar elem from the temp to
      a new temp and returns the new temp
      - `Builder::tmp` or `Builder::def` calls `program->allocateTmp` to
        allocate a new tmp
  - `Builder::copy` creates and inserts a `p_parallelcopy` instruction
    - `aco::create_instruction` creates an `Instruction` with 1 def and 1 op
    - `instr->definitions[0]` and `instr->operands[0]` are initialized
    - `Builder::insert` adds the instr to the end of the block
- `struct isel_context`
  - `allocated_vec` tracks temps that are vectors
    - e.g., `nir_op_vec2` creates a vector temp from scalar temps
    - `allocated_vec[dst.id()]` returns the scalar temps from the vector temp
    - this is used to avoid excessive copies, which could happen when
      extracting vectors from vectors if there was no `allocated_vec`

## ACO IR Pesudo Opcodes

- `aco_opcode::p_parallelcopy`
  - it copies n operands to n dsts 1:1
    - each src and dst can have any RC; the only requirement is that each pair
      must have the same size
    - if a dst is sgpr, its src must not be vgpr
  - `bld.copy` uses this opcode
  - it can load a const to a temp, or convert from sgpr to vgpr
    - `s1: %1 = p_parallelcopy 3`
    - `v1: %2 = p_parallelcopy %1`
- `aco_opcode::p_create_vector`
  - it is conceptually a `writev`
    - all operands are copied to the same dst consecutively
    - dst and operands can have any RC; the only requirement is that the dst
      size must match the sum of operand sizes
    - if dst is sgpr, operands must not be vgpr
  - it can create a vector
    - `s3: %1 = p_create_vector 2, 1, 0`
  - it can also work like this
    - `v2b: %1 = p_parallelcopy 1`
    - `v2b: %2 = p_parallelcopy 2`
    - `v1: %3 = p_create_vector %1, %2`
- `aco_opcode::p_extract_vector`
  - it copies an element from an array
    - src is considered an array whose element type is determined by dst
    - this copies a single element from src to dst
  - `emit_extract_vector` uses this opcode
  - it can convert a scalar to a vector
    - `v1: %1 = p_parallelcopy 3`
    - `v2b: %2 = p_extract_vector %1, 1`
- `aco_opcode::p_split_vector`
  - it is conceptually a `readv`
    - src is copied to dsts consecutively
    - src and dsts can have any RC; the only requirement is that the src size
      must match the sum of dst sizes
    - if any dst is sgpr, src must not be vgpr
  - `emit_split_vector` uses this opcode
  - it can split a vector into scalars
    - `s2: %3 = s_load_dwordx2 %1, %2`
    - `s1: %4,  s1: %5 = p_split_vector %3`
  - it can also split a scalar into vectors
    - `v1: %1 = p_parallelcopy 0`
    - `v2b: %2,  v2b: %3 = p_split_vector %1`
- `aco_opcode::p_as_uniform`
  - `bld.as_uniform` uses this opcode
  - it gets translated to `aco_opcode::v_readfirstlane_b32`

## Push Constants

- `visit_load_push_constant` lowers `nir_intrinsic_load_push_constant`
- if the push constant is inlined,
  - `ctx->args->inline_push_consts` defines how it is inlined
  - `aco_opcode::p_create_vector` creates the vector
    - e.g., when the type is `vec2`, it is pushed as two `s1` regs.  aco needs
      to create an `s2` reg from the two `s1` regs
- if the push constant is not inlined,
  - `ctx->args->push_constants` defines the location of the push constant
    block
  - `convert_pointer_to_64_bit` converts the 32-bit location to 64-bit addr
    using `aco_opcode::p_create_vector`
    - `ctx->options->address32_hi` is `0xffff8000`
  - `aco_opcode::s_load_dword*` loads

## System Values

- `nir_intrinsic_load_workgroup_id`
  - `aco_opcode::p_create_vector` creates an `s3` reg from
    `ctx->args->workgroup_ids`
  - `aco_opcode::p_split_vector` splits the `s3` reg into 3 `s1` regs
- `nir_intrinsic_load_local_invocation_id`
  - `aco_opcode::p_parallelcopy` copies `ctx->args->local_invocation_ids` from
    a `v3` reg to a `v3` reg
  - `aco_opcode::p_split_vector` splits the `v3` reg into 3 `v3` regs

## SSBO

- `nir_lower_explicit_io` lowers derefs to `store_ssbo`
  - `32x3  %1 = @vulkan_resource_index (%0 (0x0)) (desc_set=0, binding=0, desc_type=SSBO)`
  - `32x3  %2 = @load_vulkan_descriptor (%1) (desc_type=SSBO)`
  - `32    %3 = mov %2.z`
  - `32x2  %4 = vec2 %2.x, %2.y`
  - `@store_ssbo (%0 (0x0), %4, %3) (wrmask=x, access=writeonly, align_mul=1073741824, align_offset=0)`
- `radv_nir_apply_pipeline_layout` lowers them to, after optimizations,
  - `con 32    %0 = @load_scalar_arg_amd (base=1, arg_upper_bound_u32_amd=0)`
  - `con 32    %1 = load_const (0xffff8000 = -32768 = 4294934528)`
  - `con 64    %2 = pack_64_2x32_split %0, %1 (0xffff8000)`
  - `con 32    %3 = load_const (0x00000000)`
  - `con 32x4  %4 = @load_smem_amd (%2, %3 (0x0)) (align_mul=16, align_offset=0)`
  - `@store_ssbo (%3 (0x0), %4, %3 (0x0)) (wrmask=x, access=writeonly, align_mul=1073741824, align_offset=0)`
- `radv_nir_apply_pipeline_layout`
  - `visit_vulkan_resource_index` lowers `nir_intrinsic_vulkan_resource_index`
    - `nir_load_scalar_arg_amd` loads the set addr
    - the returned vec3 is replaced by `(set_ptr, binding_ptr, stride)`
  - `visit_load_vulkan_descriptor` lowers `nir_intrinsic_load_vulkan_descriptor`
    - the returned vec3 is replaced by `(set_ptr, binding_ptr, 0)`
  - `load_buffer_descriptor` lowers` nir_intrinsic_store_ssbo`
    - `convert_pointer_to_64_bit` returns the 64b addr of the descriptor
      - `address32_hi` is `0xffff8000`
    - `nir_load_smem_amd` loads the descriptor
    - `@store_ssbo` is kept
- `visit_intrinsic` lowers `load_scalar_arg_amd`
  - `aco_opcode::p_parallelcopy` copies from `s1` to `s1`
  - no `aco_opcode::p_split_vector` needed
- `visit_load_smem` lowers `load_smem_amd` to `aco_opcode::s_load_dwordx4`
  - it loads the 4-dword descriptor from the descriptor set
- `visit_store_ssbo` lowers `store_ssbo` to `aco_opcode::buffer_store_dword`

## Loads

- `emit_load` emits load instructions
  - `LoadEmitInfo` describes the load
    - `offset` is an `Operand` which refers to a `Temp` or a immed value
    - `const_offset` is an immed offset
    - `soffset` is a `Temp` that holds yet another offset in sgpr
    - `dst` is a `Temp`
    - `num_components` is the number of components to load
    - `component_size` is the size of a component in bytes
    - `resource` is a `Temp` that holds the buffer resource descriptor
    - `idx` is a `Temp` that holds the buffer index
      - the descriptor defines a base and a stride
      - the addr is `base + stride * idx + offset`
    - `align_mul` and `align_offset` have the same meanings as nir intrinsic
      alignment
      - see `NIR_INTRINSIC_ALIGN_MUL` and `NIR_INTRINSIC_ALIGN_OFFSET`
      - this is a guarantee on the address alignment
    - when the buffer is swizzled (tiled),
      - `split_by_component_stride` defaults to true and emits multiple load
        instructions to load a component at a time
      - `component_stride` is the stride between components
      - `swizzle_component_size` is the size of a component
    - more
  - `EmitLoadParameters` describes the limitations of the load instruction
    - `callback` is a function that emits the load instruction
      - `align` is the alignment of the effective load addr with all things
        considered
    - `byte_align_loads` is true if the load instruction requires
      dword-aligned addr for dword+ loads
      - eg, when reading 4 bytes from offset 1, the load instruction will read
        8 bytes from offset 0; the result needs to be right-shifted
    - `supports_8bit_16bit_loads` is true if the load instruction supports 8b
      and 16b loads
      - smem is the only load instruction that does not support 8b/16b loads
      - 8b and 16b loads are naturally aligned
    - `max_const_offset_plus_one` is the max supported immed offset
    - pre-defined `EmitLoadParameters`
      - `lds_load_params` false
      - `smem_load_params` true
      - `mubuf_load_params` true
      - `mtbuf_load_params` false
      - `scratch_mubuf_load_params` false
      - `scratch_flat_load_params` false
      - `global_load_params` true

## fp16

- example: self-multiplication of `f16vec2`
  - nir

    con 16x2  %3 = vec2 %1, %2
    con 16x2  %4 = fmul %3, %3
    con 16    %5 = mov %4.x
    con 16    %6 = mov %4.y
  - aco

    s1: %3 = s_pack_ll_b32_b16 %1, %2
    s1: %4 = p_parallelcopy %3
    v1: %5 = p_parallelcopy %3
    v1: %6 = v_pk_mul_f16 %4, %5
    v2b: %7 = p_extract_vector %6, 0
    s1: %8 = p_as_uniform %7
    v2b: %9 = p_extract_vector %6, 1
    s1: %10 = p_as_uniform %9
  - `nir_op_vec2`
    - `use_s_pack` is true on gfx9+
    - `bld.sop2(aco_opcode::s_pack_ll_b32_b16)` packs the two fp16 in
      `packed[0]` and `packed[1]` and replaces `packed[0]`
    - `bld.copy(Definition(dst), packed[0])` copies from `packed[0]` to `dst`
  - `nir_op_fmul`
    - `emit_vop3p_instruction(aco_opcode::v_pk_mul_f16)` does packed fmul
      - when both src0 and src1 are both sgpr, `as_vgpr` forces src1 to vgpr
  - `nir_op_mov`
    - because src is vgpr, and has 2 components, bitsize 16, and swizzle,
      `emit_extract_vector` returns a `v2b`
    - because dst is sgpr, `bld.pseudo(aco_opcode::p_as_uniform)` does the
      conversion
  - when multiplying repeated, it is more ideal to stay in vgpr than to keep
    converting back and forth...

## int64

- `nir_lower_int64_options`
  - radv enables
    - `nir_lower_divmod64`
    - `nir_lower_iabs64`
    - `nir_lower_iadd_sat64`
    - `nir_lower_imul_2x32_64`
    - `nir_lower_imul64`
    - `nir_lower_imul_high64`
    - `nir_lower_minmax64`
  - these are not enabled
    - `nir_lower_bit_count64`
    - `nir_lower_extract64`
    - `nir_lower_find_lsb64`
    - `nir_lower_iadd64`
    - `nir_lower_icmp64`
    - `nir_lower_ineg64`
    - `nir_lower_isign64`
    - `nir_lower_logic64`
    - `nir_lower_mov64`
    - `nir_lower_scan_reduce_bitwise64`
    - `nir_lower_scan_reduce_iadd64`
    - `nir_lower_shift64`
    - `nir_lower_subgroup_shuffle64`
    - `nir_lower_ufind_msb64`
    - `nir_lower_usub_sat64`
    - `nir_lower_vote_ieq64`
- `nir_op_iadd`
  - `nir_lower_int64` would lower it with `lower_iadd64`
    - unpack, two additions with carry, repack
  - `visit_alu_instr` is similar?

## `aco::visit_cf_list`

- `aco::visit_cf_list` performs instruction selections
  - we focus on `aco::visit_if` and `aco::visit_loop` here
- there are logical cfg and linear cfg
  - logical cfg is a subset of linear cfg
  - when there are divergent control flows, extra nodes are added to linear
    cfg
    - some to avoid critical edges
    - some to invert the exec mask
- `aco::visit_if`
  - nir
    - `if %1`
    - `32    %4 = fadd %2, %3`
    - `else`
    - `32    %7 = fsub %5, %6`
  - if `%1` is uniform, the logical and linear cfgs are the same
    - `bool_to_scalar_condition`
      - `s2: %3,  s1: %2:scc = s_and_b64 %1, %0:exec` when wave64
    - `begin_uniform_if_then`
      - `p_logical_end`
      - `s2: %4 = p_cbranch_z %2:scc`
      - `BB1`
      - `p_logical_start`
    - `begin_uniform_if_else`
      - `p_logical_end`
      - `s2: %5 = p_branch`
      - `BB2`
      - `p_logical_start`
    - `end_uniform_if`
      - `p_logical_end`
      - `s2: %6 = p_branch`
      - `BB3`
      - `p_logical_start`
  - if `%1` is divergent, the logical cfg is a subset of the linear cfg
    - the exec mask has been set up
    - `begin_divergent_if_then`
      - `p_logical_end`
      - `s2: %2 = p_cbranch_z %1`
      - `BB1` (logical then)
      - `p_logical_start`
    - `begin_divergent_if_else`
      - `p_logical_end`
      - `s2: %3 = p_branch`
      - `BB2` (linear then)
      - `s2: %4 = p_branch`
      - `BB3` (linear invert, `insert_exec_mask` will invert exec mask here)
      - `s2: %5 = p_branch`
      - `BB4` (logical else)
      - `p_logical_start`
    - `end_divergent_if`
      - `p_logical_end`
      - `s2: %6 = p_branch`
      - `BB5` (linear else)
      - `s2: %7 = p_branch`
      - `BB6`
      - `p_logical_start`

## `aco::dominator_tree`

- it calculates `logical_idom` and `linear_idom` for all blocks
  - if block A's `logical_idom` points to block B, it means block B
    immediately dominates A

## `aco::lower_phis`

- when we have a phi of sgpr,
  - nir
    - `32    %1 = load_const (0)`
    - `32    %3 = phi b0: %1 (0), b1: %2`
  - instruction selection
    - `s1: %1 = p_parallelcopy 0`
    - `s1: %3 = p_phi %1, %2`
  - `aco::lower_phi_to_linear`
    - `s1: %1 = p_parallelcopy 0`
    - `s1: %3 = p_linear_phi %1, %2`
- when we have a phi of fp16,
  - nir
    - `16    %1 = load_const (0.0)`
    - `16    %3 = phi b0: %1 (0.0), b1: %2`
  - instruction selection
    - `s1: %1 = p_parallelcopy 0`
    - `v2b: %3 = p_phi %1, %2`
  - `aco::lower_subdword_phis` makes sure all operands have the same rc
    - `s1: %1 = p_parallelcopy 0`
    - `v1: %4 = p_parallelcopy %1`
    - `v2b: %5 = p_extract_vector %4, 0`
    - `v2b: %3 = p_phi %5, %2`

## `aco::value_numbering`

- it depends on dominator tree and uses value numbering to perform CSE

## `aco::optimize`

## `aco::setup_reduce_temp`

- it insert `p_start_linear_vgpr` and `p_end_linear_vgpr`

## `aco::insert_exec_mask`

- it inserts `exec_mask` maniuplation and updates branching

## `aco::live_var_analysis`

- `aco::live`
  - `live_out` is per-block `IDSet`
    - it tracks temps used by other blocks
  - `register_demand` is per-block per-instruction `RegisterDemand`
- `process_live_temps_per_block` processes blocks and instructions backward
  - `IDSet live = lives.live_out[block->index]` inits live-in from live-out
    - e.g., if the block has no instruction, then its live-in and live-out are
      equal
  - for each non-phi instruction,
    - each def is removed from live-in
    - each op is added to live-in
    - iow, if an op is not a def of the block, it belongs to live-in
    - also,
      - if a def is not in live-in, it means the def has no user in the block
        (because we traverse backward) and is marked `setKill`
      - the last op that uses the same temp is marked `setFirstKill` or
        `setKill`, because the temp has no more user in the block
  - for each phi instruction,
    - the def is removed from live-in
    - each op is added to live-out of the corresponding predecessor
  - with live-in of the current block known, we can update `live_out` of
    predecessors
    - if a predecessor is updated, `worklsit` is updated as well such that the
      predecessor will be visited again
- if called before `CompilationProgress::after_ra`, `update_vgpr_sgpr_demand`
  updates the demands
  - `sgpr_limit` and `vgpr_limit` are the hw limit
  - `program->num_waves` is updated based
    - sgpr and vgpr demands
    - hw limit (20 on rdna2+)
    - lds, etc.
  - `program->max_reg_demand` is updated based on `program->num_waves`

## `aco::spill`

- it spills if reg use exceeds hw limit

## `aco::schedule_program`

- instruction scheduling can create new sgpr/vgpr demands

## `aco::register_allocation`

- the goal is to assign a physical reg to each temp
  - we could assign a unique reg to each temp, but that's insane
    - high reg pressure
    - no dce (e.g., `p_parallelcopy` can be eliminated if src and dst are
      assigned the same reg)
  - with liveness analysis, we can recycle a reg when the associated temp is
    dead
- `ra_ctx`
  - `assignments` tracks how each temp should be assigned
    - `affinity` means a temp should be assigned the same reg as another temp
    - `vcc` means a temp shoudl be assigned special reg `vcc`
    - `m0` means a temp shoudl be assigned special reg `m0`
  - `vectors` maps operands of `p_create_vector` back to the instr
    - used for affinity
  - `split_vectors` maps operands of `p_split_vector` back to the instr
    - used for affinity too
- `get_affinities`
  - it loops over all phi and non-phi instrs to update `vectors`,
    `split_vectors`, `temp_to_phi_resources` and `phi_resources`
    - for `p_create_vector`, `vectors` is updated
    - for `p_split_vector`, `split_vectors` is updated
    - for phi, operands that are killed (consumed) are added to
      `phi_resources[idx]`, and `temp_to_phi_resources[op.id()] = idx` is
      added
    - for copy, if def is used as an operand of phi, and the operand of the
      copy is killed, the operand of copy is also added to `phi_resources`
  - at the end, it loops over `phi_resources` and updates affinity
    - `ctx.assignments[op.id()].affinity = def.id()`
  - e.g., `a = phi b` and `b` has no other user thus killed
    - `ctx.assignments[b.id()].affinity = a.id()`
    - since `a` and `b` are assigned the same reg, `phi` can be eliminated
      later
- `get_reg` returns a `PhysReg` for a `Temp`
  - if the temp is an operand of `p_split_vector`, `p_create_vector`, `p_phi`,
    `p_parallelcopy`, etc., the affinity might have been set and it should be
    honored
  - otherwise, `get_reg_impl` allocates a reg for the temp
  - if there is no more reg,
    - `compact_linear_vgprs` compacts linear vgrs and retry
    - `increase_register_file` increases reg limit and retry
    - `compact_relocate_vars` relocates all regs to make room
- for each block
  - `init_reg_file`
  - `get_regs_for_phis` assigns regs for phis
  - for each non-phi instr
    - for each operand
      - assert that an reg has been assigned, because we loop forward and the
        reg was assigned by def of some earlier instruction
      - if `operand_can_use_reg` returns false, `get_reg_for_operand` renames
        the reg
    - for each definition
      - if it needs a fixed reg, and the reg is already used,
        `get_regs_for_copies` and `update_renames`
      - if the instr is pseudo, depending on the instr, we can
        - `get_reg_specified` to use the same reg as the operand
        - `get_reg_simple`
        - `get_reg_create_vector` for `p_create_vector`
      - otherwise, `get_reg` to allocate a reg
    - `emit_parallel_copy` handles copies incurred by renames, etc.
- e.g.,
  - `BB0` is the entrypoint
    - there is no phi
    - for each non-phi instr,
      - the operands have been assigned
      - the definitions need assignments
  - `BB1` is the loop header
    - phis need assignments
      - definitions usually need assignments
      - operands are usually have affinities to the definitions
    - non-phi definitions need assignment
  - `BB2` is the loop body
    - similar to `BB0`
  - `BB3` is the loop exit
    - phis need assignments
      - definitions usually just get the regs from the operands
    - non-phi definitions need assignment

## `aco::optimize_postRA`

## `aco::ssa_elimination`

- it removes empty blocks and inserts copies such that phis can be ignored

## `aco::lower_to_hw_instr`

- it lowers pseudo instrs to hw instrs
- `handle_operands` are called to instrs that copy
  - it is used for
    - `p_extract_vector`
    - `p_create_vector`
    - `p_start_linear_vgpr`
    - `p_split_vector`
    - `p_parallelcopy`
    - `p_as_uniform`
  - `copy_map` describes the `copy_operation` for each reg
    - e.g., `b = p_parallelcopy a`
      - the reg is `b.physReg()`
      - `def` is `b`
      - `op` is `a`
      - `bytes` is `a.bytes()`
    - if `a` and `b` have the same reg, the copy can be skipped
    - `try_coalesce_copies` tries to coalesce copies
  - `do_copy` emits the instruction to copy
  - if the copies form a cycle, `do_swap` breaks the cycle
    - it uses xor to swap, <https://en.wikipedia.org/wiki/XOR_swap_algorithm>

## `aco::schedule_vopd` and `aco::schedule_ilp`

- they schedules instrs for instruction level parallelism

## `aco::insert_wait_states`

- it inserts `aco_opcode::s_waitcnt`, `aco_opcode::s_waitcnt_vscnt`, and
  `aco_opcode::s_delay_alu` for correctness

## `aco::insert_NOPs`

- it inserts dummy nops and other instrs to work around hw bugs

## `aco::form_hard_clauses`

- it inserts `aco_opcode::s_clause` to improve performance
