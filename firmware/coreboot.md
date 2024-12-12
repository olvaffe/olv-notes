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
- example
  - `coreboot.pre` is created with the size specified by `fmap.fmap`
    - for each CBFS section, the CBFS header is generated
    - for each file to be added to CBFS
      - find the CBFS header and memcpy the file into `coreboot.pre`
  - when there are other sections, for proprietary code,
    - `SI_DESC` section is dd'ed from a `descriptor.bin`; this makes
      `coreboot.pre` an IFDT image
    - the other sections, such as `SI_ME` are patched into `coreboot.pre` from
      `me.bin` using ifdttool

## bootblock

- take intel for example
- when CPU comes out of reset, it executes from `_X86_RESET_VECTOR`
  - `_X86_RESET_VECTOR` is `0xfffffff0`
  - this is the first instruction the CPU fetches and executes after reset
  - well, except for FIT (Firmware Interface Table)
- `src/arch/x86/bootblock.ld` puts `.reset` section at `_X86_RESET_VECTOR`
  - `src/cpu/x86/reset16.S` defines the `.reset` section
  - the cpu starts executing from `_start` which consists of a single
    instruction to jump to `_start16bit` defined in `src/cpu/x86/entry16.S`
- `_start16bit` does a bit of initialization and enters the protected mode by
  jumping to `bootblock_protected_mode_entry` in `src/cpu/x86/entry32.S`
- after a few more intialization, it jumps to `bootblock_pre_c_entry` in
  `src/soc/intel/common/block/cpu/car/cache_as_ram.S`
- after setting up cache-as-ram, it jumps to `car_init_done` to set up the
  stack and calls `bootblock_c_entry`.
- the soc-specific C entry `bootblock_c_entry` calls the common
  `bootblock_main_with_basetime`
  - `bootblock_soc_early_init`
    - initialize early pch, spi
  - `bootblock_mainboard_early_init`
    - e.g., initialize early gpio
  - `bootblock_soc_init`
    - initialize pch
  - `bootblock_mainboard_init`
    - e.g., initialize gpio
  - `run_romstage`
- `run_romstage` is defined in `src/lib/prog_loaders.c`
  - load and decompress romstage from cbfs
  - call the entry point of romstage

## romstage

- take intel for example
- bootblock calls the romstage
- the romstage entry point is `_start` defined in `src/arch/x86/assembly_entry.S`
  - it calls `car_stage_entry`, which calls `romstage_main`
- after initializing dram, `romstage_main` calls `run_postcar_phase` to enter
  the postcar stage
  - `vboot_run_logic` first to optionally enter verstage
    - `verstage_main`
      - `vb2api_fw_phase1`
      - `vb2api_fw_phase2`
      - `vb2api_fw_phase3`
    - `after_verstage`
  - it loads and decompresses postcar stage from cbfs
  - call the entry point of postcar stage
- the entry of the postcar stage is `_start` of `src/arch/x86/exit_car.S`
- postcar tears down cache-as-ram and calls `main` in `src/arch/x86/postcar.c`
- `run_ramstage` loads, decompresses, and enters the ramstage

## ramstage

- take intel for example
- romstage calls `_start` in `src/arch/x86/c_start.S` which calls `main` in
  `src/lib/hardwaremain.c`
- after walking through states in `boot_states`, `bs_payload_load` loads the
  payload and `bs_payload_boot` jumps to the the payload

## payload

- seabios for traditional bios
- tianocore for UEFI
- depthcharge for Chromebook

## Intel ucode, FSP, and ME

- ucode can be updated through FIT, Firmware Interface Table
- FSP provides proprietary code that can be integrated into various stages to
  perform the tasks

## Flashmap

- `.fmd` describe how the flash (usually NOR flash) is paritioned
- take intel for example
  - usually starts with `FLASH@0xfe000000 0x2000000` for a 32MB NOR flash at
    the top of the 4GB address space
  - [0MB..5MB]
    - 4KB `SI_DESC` for Intel Flash Descriptor
    - the rest is `SI_ME` for Intel ME
  - [5MB..20MB] is for legacy CBFS
  - [20MB..28MB] is for RW
  - [28MB..32MB] is for coreboot.rom
- <https://github.com/coreboot/coreboot/blob/main/src/mainboard/google/rauru/chromeos.fmd>
  - 8MB flash
    - [0MB..4MB] `WP_RO`
    - [4MB..5.5MB] `RW_SECTION_A` and `RW_MISC`
    - [5.5MB..7MB] `RW_SECTION_B` and `RW_SHARED`
    - [7MB..8MB] `RW_LEGACY`
  - `cbfstool <ap-fw.bin> layout -w` shows the same info
  - `cbfstool <ap-fw.bin> print [-r COREBOOT]` shows the contents of `COREBOOT`
    CBFS region

## vboot

- verified boot
- divide the SPI flahs into 4 sections
  - read-only section
  - read-only GBB (Google Binary Blob) area
  - read-write section A
  - read-write section B
- read-only section has
  - VPD (Vital Product Data) area
  - firmware id area
  - recovery firmware
  - bootblock
- read-only GBB section has
  - root public RSA key
  - more
- read-write sections have
  - `VBLOCK` subsection
    - verified using the root key
    - extract the firmware signing key to verify `FW_MAIN`
  - `FW_MAIN` subsection contains
    - romstage, ramstage, and payload
