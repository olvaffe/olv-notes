# ARM64 MM

## ARM32

- Kconfig
  - `FLATMEM`
- In `setup_arch`,
  - `paging_init` sets up the page tables
    - `map_lowmem` creates the linear map

## ARM64

- Kconfig
  - `SPARSEMEM_VMEMMAP`
- In `setup_arch`,
  - `paging_init` sets up the page tables
