ARM64
=====

## Bootloader

- bootloader has four responsibilities
  - initialize RAM
  - load device tree blob (.dtb)
  - load decompressed kernel image
  - jump to kernel image
- the kernel image has a 64-byte header
  - the first 8 bytes are instructions to jump to stext
- before jumping to the kernel,
  - reg x0 must contain the address of the loaded dtb
  - MMU must be off
  - more
- bootloader can patch the dtb
  - for serial number
  - machine revision
  - reserved memory
  - etc

## Booting

- `kernel/head.S`
  - `_head` defined in `head.S` defines the 64-byte image header
  - the first instruction is `b primary_entry`
  - after some setup and enabling MMU, it jumps to `start_kernel`
  - dtb address is saved to `__fdt_pointer`
- `start_kernel`
  - `smp_setup_processor_id` prints "Booting Linux on physical CPU ..."
  - `linux_banner` is printed ("Linux version ...")
  - `setup_arch` is arch-specific
    - `setup_machine_fdt` prints "Machine model: ..."
    - `efi_init` prints "efi: UEFI not found.", if `CONFIG_EFI`
    - `arm64_memblock_init` prints "Reserved memory: create CMA memory pool at ..."
    - more
  - more

## Flattened Device Tree (FDT)

- `setup_machine_fdt` passes dtb addr to `setup_machine_fdt`
- `early_init_fdt_scan_reserved_mem` scans `/memreserve/` header or
  `reserved-memory` property and reserves them from memblock
  - `fdt_init_reserved_mem` makes reserved memory available to drivers
  - for example, `rmem_cma_setup` is called for memory reserved for
    `shared-dma-pool`
