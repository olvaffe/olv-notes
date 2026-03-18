Linux `init_mm`
===============

## Overview

- `init_mm` is the mm of `init_task`
- `init_mm.pgd` is `swapper_pg_dir`

## Kernel Image Mapping

- arm
  - the goal is to map kernel image to `KIMAGE_VADDR`
  - `primary_entry`
    - `create_init_idmap` inits `__pi_init_idmap_pg_dir`
      - the symbol is prefixed by `--prefix-symbols=__pi_`
      - it creates identity mapping for
        - `[_stext, __initdata_begin]` with `PAGE_KERNEL_ROX`
        - `[__initdata_begin, _end]` with `PAGE_KERNEL`
      - the goal is such that enabling mmu does not result in fault
        immediately
      - `ptep` points to "the next available page"
        - the pgtable hierarchy needs multiple pgtables
        - when `map_range` needs a new one, it uses `ptep` and increments it
          by a page
        - `vmlinux.lds.S` reserves `INIT_IDMAP_DIR_SIZE`
  - `__primary_switch`
    - `__enable_mmu`
      - it sets `ttbr0_el1` to `__pi_init_idmap_pg_dir`
        - ttbr0 is for userspace mapping; it changes on context switch
        - but we use it for idmap for now
      - it sets `ttbr1_el1` to `reserved_pg_dir`
        - ttbr1 is for kernel mapping; it does not change on context switch
        - `reserved_pg_dir` is always empty
    - `early_map_kernel` maps the kernel image to `KIMAGE_VADDR`
      - it inits `__pi_init_pg_dir` to map kernel image to `KIMAGE_VADDR`
        - `[_text, _stext]` with `PAGE_KERNEL`
        - `[_stext, _etext]` with `PAGE_KERNEL`
        - `[__start_rodata, __inittext_begin]` with `PAGE_KERNEL`
        - `[__inittext_begin, __inittext_end]` with `PAGE_KERNEL`
        - `[__initdata_begin, __initdata_end]` with `PAGE_KERNEL`
        - `[_data, _end]` with `PAGE_KERNEL`
      - `pgdp` points to "the next available page"
        - similar to `create_init_idmap`, we need multiple pgtables for levels
        - `vmlinux.lds.S` reserves `INIT_DIR_SIZE`
      - `idmap_cpu_replace_ttbr1` replaces `reserved_pg_dir` by
        `__pi_init_pg_dir`
        - both pa (translated by `__pi_init_idmap_pg_dir`) and va (translated
          by `__pi_init_pg_dir`) work
      - it copies `init_pg_dir` to `swapper_pg_dir`
      - `idmap_cpu_replace_ttbr1` replaces `__pi_init_pg_dir` by
        `swapper_pg_dir`
  - `tramp_pg_dir`
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

- arm
  - the goal is to map all physical memory to `PAGE_OFFSET`
  - `setup_arch` calls `paging_init`
    - `map_mem` creates the linear mapping for physical ram
      - `for_each_mem_range` loops for memblocks and `__map_memblock` maps
        them
      - it maps pa of physical ram to `__phys_to_virt(pa)`
        - that is, `pa - PHYS_OFFSET + PAGE_OFFSET`
        - `PHYS_OFFSET` is the first of physical ram
    - `create_idmap` creates the identity mapping for `.idmap.text` section
      - it uses `idmap_pg_dir` rather than `__pi_init_idmap_pg_dir`
      - the storage is from `idmap_ptes` rather than `INIT_IDMAP_DIR_SIZE`
      - it will be used briefly during mmu state transition
        - e.g., between `cpu_install_idmap` and `cpu_uninstall_idmap`
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

- the goal is to allow the kernel space to access both kernel mappings and
  userspace mappings at the same time
- arm
  - `ttbr0_el1` points to user pgd for userspace mappings
    - `context_switch` calls `switch_mm_irqs_off` before `switch_to`
    - `__switch_mm`
      - if `init_mm`, `cpu_set_reserved_ttbr0` installs `reserved_pg_dir`
      - else, `check_and_switch_context` calls `cpu_switch_mm` to install
        `mm->pgd`
  - `ttbr1_el1` always points to `swapper_pg_dir` for kernel mappings
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

## Relocatable

- ELF
  - each `PT_LOAD` segment has `p_paddr` and `p_vaddr`
    - `p_paddr` is the physical address to load to
      - it is ignored by the loader if the loader expects mmu
      - it can be ignored to some degree if the code is PI
    - `p_vaddr` is the virtual address to map to
      - it can be ignored to some degree if the code is PI
  - on x86, there is a `PT_LOAD` with
    - `p_paddr` is `0x1000000`
    - `p_vaddr` is `0xffffffff81000000`
  - on arm, there is a `PT_LOAD` with
    - `p_paddr` is `0xffff800080000000`
    - `p_vaddr` is `0xffff800080000000`
- arm
  - the kernel image can always be loaded to any physical address
  - `vmlinux.lds.S` sets the base va to `KIMAGE_VADDR`
    - `PAGE_OFFSET` is   `0xfff0000000000000`
    - `MODULES_VADDR` is `0xffff800000000000`
    - `KIMAGE_VADDR` is  `0xffff800080000000`
  - `CONFIG_RELOCATABLE` allows the kernel to be mapped to another address
    other than `KIMAGE_VADDR`
    - this is possible because `relocate_kernel` patches in relocs
  - `CONFIG_RANDOMIZE_BASE` maps the kernel to another address
    - `early_map_kernel` calculates `kaslr_offset` and maps the kernel to
      `KIMAGE_VADDR + kaslr_offset`
- x86
  - the bzimage can always be loaded to any physical address
    - `extract_kernel` calls `decompress_kernel` to decompress to
      `CONFIG_PHYSICAL_START`, unless relocatable
  - `vmlinux.lds.S` sets the base va to `__START_KERNEL`, which is
    `__START_KERNEL_map` plus `CONFIG_PHYSICAL_START`
    - `LOAD_OFFSET` is `__START_KERNEL_map`
      - this is why `p_paddr = p_vaddr - LOAD_OFFSET`
    - `PAGE_OFFSET` is        `0xff11000000000000`
    - `__START_KERNEL_map` is `0xffffffff80000000`
    - `MODULES_VADDR` is      `0xffffffffa0000000`
  - `CONFIG_RELOCATABLE` allows the kernel to be decompressed to another
    address other than `CONFIG_PHYSICAL_START` and mapped to another address
    other than `__START_KERNEL_map`
    - `extract_kernel` calls `decompress_kernel` with a different output
      address
    - `virt_addr` is an offset from `__START_KERNEL_map`
      - `handle_relocations` patches in relocs
  - `CONFIG_RANDOMIZE_BASE`
    - `choose_random_location` randomises both `output` and `virt_addr`
