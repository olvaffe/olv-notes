Mesa RADV ACO
=============

## ACO

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
