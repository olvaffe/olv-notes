Linux `init_mm`
===============

## Overview

- `init_mm` is the mm of `init_task`
- `init_mm.pgd` is `swapper_pg_dir`

## Kernel Image Mapping

- x86
  - the goal is to map kernel image to `__START_KERNEL_map`
  - `startup_64`
    - `call __pi___startup_64` calls `__startup_64`
      - the symbol is prefixed by `--prefix-symbols=__pi_`
      - it initializes `early_top_pgt`
      - `p2v_offset` is an offset from compile-time va to runtime pa
        - `leaq common_startup_64(%rip), %rdi` computes pa of `common_startup_64`
          - at compile-time, assembler resolves `common_startup_64` to an
            offset from the current location; e.g., -60 if it is -60 bytes
            before `lea`
          - at runtime, it computes pa
        - `subq .Lcommon_startup_64(%rip), %rdi`
          - `SYM_DATA_LOCAL(.Lcommon_startup_64, .quad common_startup_64)`
            saves compile-time va of `common_startup_64` at the addr
          - at runtime, it dereferences to the compile-time va and subtracts
            it from runtime pa
      - `physaddr` is the runtime pa of `_text`, kernel image start
      - `la57` is true for 5-level paging
      - `__START_KERNEL_map` is the compile-time va of kernel region (to be mapped)
      - `load_delta` is runtime pa of the kernel region
      - `va_text` is the compile-time va of `_text`
      - `va_end` is the compile-time va of `_end`
      - after the initializatoin, `early_top_pgt` has
        - identity map for the kernel image
        - `__START_KERNEL_map` to pa map
    - switches `cr3` to `early_top_pgt`
      - the identity map keeps this working
  - `x86_64_start_kernel`
    - `reset_early_page_tables` tears down identity map from `early_top_pgt`
      - now that we call `x86_64_start_kernel`, we no longer need the identity
        map
    - `clear_page(init_top_pgt)` zeros `init_top_pgt`
    - `init_top_pgt[511] = early_top_pgt[511]` copies the kernel region

## Memory Linear Mapping

- x86
  - the goal is to map all physical memory to `PAGE_OFFSET`
  - note that `swapper_pg_dir` defines to `init_top_pgt`
    - there is an existing `__START_KERNEL_map` mapping
  - `setup_arch` calls `init_mem_mapping`
    - `init_memory_mapping` maps `[0, ISA_END_ADDRESS]` to linear mapping
      - `kernel_physical_mapping_init` sets up pgtable
    - `memory_map_top_down` maps `[ISA_END_ADDRESS, max_pfn]` to linear
      mapping
      - `init_range_memory_mapping` maps each memblock reported by e820
    - `load_cr3(swapper_pg_dir)` switches to `swapper_pg_dir`

## User MM

- x86
  - when `mm_alloc_pgd` calls `pgd_alloc` to allocate a pgd for a user mm,
    - `pgd_ctor` calls `clone_pgd_range` to clone all kernel mappings from
      `swapper_pg_dir` to user pgd
      - this is fine because those mappings are `PAGE_KERNEL` and lack
        `PTE_USER`
      - on the other hand, userspace mmap looks up their prot in
        `vm_get_page_prot` which includes `PTE_USER` mostly
  - when any kernel mapping changes, `arch_sync_kernel_mappings` propagates
    the change from `swapper_pg_dir` to all user pgds
