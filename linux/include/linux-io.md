Kernel IO
=========

## `ioremap`

- `ioremap` maps physical addr range for mmio
  - arm
    - `pfn_is_map_memory` calls `memblock_is_map_memory` and rejects system ram
    - `generic_ioremap_prot`
      - `__get_vm_area_caller` allocs a vm area
      - `ioremap_page_range` sets up pgtable
  - x86
    - `__ioremap_check_mem` calls `walk_mem_res` and rejects system ram
    - `get_vm_area_caller` allocs a vm area
    - `ioremap_page_range` sets up pgtable

## IO

- `readl`
  - arm
    - `__io_br` expands to nop
    - `__raw_readl` expand to `ldr`
    - `__io_ar` expands to `volatile("dmb oshld":::"memory")` and a control dep
  - x86
    - `volatile("movl ...":...:"memory")`
    - `memory` clobber is a compiler barrier
    - ther is no memory barrier
      - the memory is mapped UC/WC which prevents hw reodering
- `writel`
  - arm
    - `__io_bw` expands to `volatile("dmb oshst":::"memory")`
    - `__raw_writel` expand to `str`
    - `__io_aw` expands to nop
  - x86: similar to `readl`
