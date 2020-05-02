coreboot
========

## Build System

- bootblock
  - build `bootblock.debug`
    - `$(objcbfs)/bootblock.debug: $$(bootblock-objs)`
    - use `memlayout.ld` as the link script
  - create `bootblock.elf`
    - `$(objcbfs)/%.elf: $(objcbfs)/%.debug`
    - strip out debug info
  - create `bootblock.raw.elf`
    - `$(objcbfs)/bootblock.raw.elf: $(objcbfs)/bootblock.elf`
    - mostly for ARM with `CONFIG_COMPRESS_BOOTBLOCK`?
  - create `bootblock.raw.bin`
    - `$(objcbfs)/bootblock.raw.bin: $(objcbfs)/bootblock.raw.elf`
    - `objcopy -O binary`
  - create `bootblock.bin`
    - `$(objcbfs)/%.bin: $(objcbfs)/%.raw.bin`
    - `cp` for most socs
- other stages stop at `.elf` because there is an ELF loader
  - `$(objcbfs)/romstage.debug: $$(romstage-objs)`
  - `$(objcbfs)/ramstage.debug: $$(ramstage-objs)`
  - `$(objcbfs)/%.elf: $(objcbfs)/%.debug`
- final image
  - `$(obj)/coreboot.pre: $(objcbfs)/bootblock.bin $$(prebuilt-files) $(CBFSTOOL) $(IFITTOOL) $$(cpu_ucode_cbfs_file) $(obj)/fmap.fmap $(obj)/fmap.desc`
    - `CONFIG_FMDFILE` specifies the `.fmd` file from whcih `fmap.fmap` is
      generated
  - `$(obj)/coreboot.rom: $(obj)/coreboot.pre $(RAMSTAGE) $(CBFSTOOL) $$(INTERMEDIATE)`
  - for each file in `cbfs-files-y`,
    - it is an file to be created inside cbfs
    - $(file)-file specifies which on-disk file to copy from

## bootblock

- take x86 for example
- the entry point is defined by `arch/x86/bootblock_crt0.S`
- when CPU is reset, it executes from `CONFIG_X86_RESET_VECTOR`
- the reset vector points to `_start` in `cpu/x86/16bit/reset16.inc`, and it
  jumps to `_start16bit` in `cpu/x86/16bit/entry16.inc`
- `_start16bit` does a bit of initialization and enters the protected mode by
  jumping to `__protected_start` in `cpu/x86/32bit/entry32.inc`
- after a few more intialization, it jumps to `bootblock_pre_c_entry` in
  `soc/intel/common/block/cpu/car/cache_as_ram.S`
- after setting up cache-as-ram, it jumps to `car_init_done` to set up the
  stack and calls `bootblock_c_entry`.  The soc-specific C entry calls the
  common `bootblock_main_with_basetime`
- `bootblock_soc_early_init`
  - initialize early pch, spi
- `bootblock_mainboard_early_init`
- `bootblock_soc_init`
  - initialize pch
- `bootblock_mainboard_init`
- `run_romstage`
  - load and decompress romstage from cbfs
  - call the entry point of romstage

## romstage

- take x86 for example
- bootblock calls `_start` in `arch/x86/assembly_entry.S`, which calls
  `car_stage_entry`, which calls `romstage_main`
- after initializing dram, `run_postcar_phase` is called to enter the postcar
  stage
- the entry of the postcar stage is `_start` of `arch/x86/exit_car.S`
- postcar tears down cache-as-ram and calls `main` in `arch/x86/postcar.c`
- `run_ramstage` loads, decompresses, and enters the ramstage

## ramstage

- take x86 for example
- romstage calls `_start` in `src/arch/x86/c_start.S` which calls `main` in
  `src/lib/hardwaremain.c`
- after walking through states in `boot_states`, it loads the payload and
  enters the payload

## payload

- seabios for traditional bios
- tianocore for UEFI
- depthcharge for Chromebook

## vboot

- verified boot
- divide the SPI flahs into 4 sections
  - read-only section
  - GBB (Google Binary Blob) area
  - read-write section A
  - read-write section B
- read-only section has
  - VPD (Vital Product Data) area
  - firmware id area
  - recovery firmware
- GBB section has
  - RSA key
  - more
