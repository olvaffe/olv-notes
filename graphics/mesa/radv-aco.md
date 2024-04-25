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
    - `aco::optimize` performs pre-RA optimizations
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
- `aco::init_context` initializes isel context
  - `nir_divergence_analysis` marks non-uniform ssas as `divergent`
  - it loops over all nir instructions to intialize
    `ctx->program->temp_rc.data`
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
- `aco::visit_alu_instr` selects the aco instruction for a `nir_alu_instr`

## ACO IR Pesudo Opcodes

- `aco_opcode::p_parallelcopy`
  - `bld.copy` uses this opcode
- `aco_opcode::p_create_vector`
- `aco_opcode::p_extract_vector`
  - `emit_extract_vector` uses this opcode
- `aco_opcode::p_split_vector`
  - `emit_split_vector` uses this opcode
- `aco_opcode::p_as_uniform`
  - `bld.as_uniform` uses this opcode
  - it gets translated to `aco_opcode::v_readfirstlane_b32`

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

## `aco::lower_to_hw_instr`

- `p_parallelcopy` requires `handle_operands` which calls `do_swap`
  - it uses xor to swap, <https://en.wikipedia.org/wiki/XOR_swap_algorithm>
