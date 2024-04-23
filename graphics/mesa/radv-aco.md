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
