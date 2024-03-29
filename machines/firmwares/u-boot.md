Das U-Boot
==========

## Overview

- <https://source.denx.de/u-boot/u-boot.git>
- <https://docs.u-boot.org/en/latest/develop/release_cycle.html>
  - quarterly release
  - a release is tagged on the first Monday of each quarter
  - then the merge window is opened for 3 weeks

## Build

- <https://docs.u-boot.org/en/latest/build/gcc.html>
  - `make nanopi-r5c-rk3568_defconfig`
  - `make menuconfig`
  - `make CROSS_COMPILE=aarch64-linux-gnu-`
- <https://docs.u-boot.org/en/latest/board/rockchip/rockchip.html>
  - build or get tf-a firmware
    - <https://github.com/ARM-software/arm-trusted-firmware.git>
    - <https://github.com/rockchip-linux/rkbin>
  - get tpl firmware
    - also at <https://github.com/rockchip-linux/rkbin>
  - `make CROSS_COMPILE=aarch64-linux-gnu- BL31=$rkbin/bin/rk35/rk3568_bl31_v1.43.elf ROCKCHIP_TPL=$rkbin/bin/rk35/rk3568_ddr_1560MHz_v1.18.bin`
    - `idbloader.img` is to be flashed to sector 0x40
      - this is mandated by the bootrom
    - `u-boot.itb` is to be flashed to sector 0x4000
      - because `CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR` defaults to `0x4000`
        for `CONFIG_ARCH_ROCKCHIP`

## CLI

- `bdinfo` prints board info
- boot variants
  - `boot` and `bootd` run `bootcmd` env
  - `bootefi` boots an efi payload
  - `bootelf` boots an elf image
  - `booti` boots a kernel image
  - `bootm` boots an app image
  - `bootp` boots from pxe
  - `bootflow scan` boots boot flows
- `coninfo` prints console info
- `env` handles env
- load variants
  - `ext2load` loads a file from ext2
  - `ext4load` loads a file from ext4
  - `fatload` loads a file from fat
  - `load` loads a file from any fs?
- `ls` lists a fs
- `mmc` handles mmc
- `net list` lists net devices
- `part` handles partitions
- `pci` lists pci devices
- `run` runs cmds in an env var
- `usb` handles usb
- `version` shows version

## Code Flow: armv8

- `_start:` in `arch/arm/cpu/armv8/start.S`
  - soc-specific `boot0.h` branches to `reset`
  - `reset` branches to `_main`
- `ENTRY(_main)` in `arch/arm/lib/crt0_64.S`
  - calls `board_init_f` in `common/board_f.c`
    - it calls the init functions on `init_sequence_f` array
  - branches to `board_init_r` in `common/board_r.c`
    - it calls the init functions on `init_sequence_r` array
    - some `init_sequence_r` init functions
      - `initr_net` calls `eth_initialize` to initialize NICs
        - console: `Net:   eth0: <name>`
      - `run_main_loop` is the last function and calls `main_loop`
- `main_loop` in `common/main.c`
  - `run_preboot_environment_command` runs `CONFIG_PREBOOT` commands
    - it runs `usb start` by default
      - console: `starting USB...`
      - `do_usb_start` calls `usb_init` to initialize and scan usb
  - `bootdelay_process` gets boot cmd
    - boot delay defaults to `CONFIG_BOOTDELAY`, which is 2 seconds
    - it returns `bootcmd` env which defaults to `CONFIG_BOOTCOMMAND`, which
      is `bootflow scan`
  - `autoboot_command` autoboots
    - `abortboot` calls `abortboot_single_key` to wait for autoboot abort
      - console: `Hit any key to stop autoboot:  <seconds>`
    - `run_command_list` runs the commands
  - `cli_loop` prompts for `CONFIG_SYS_PROMPT` for interactive shell

## Standard Boot

- <https://docs.u-boot.org/en/stable/develop/bootstd.html>
- configs
  - `CONFIG_BOOTSTD` is enabled by default to show the config items
  - `CONFIG_BOOTSTD_DEFAULTS` enables commonly needed features
    - it enables `CONFIG_BOOTMETH_EXTLINUX` and `CONFIG_BOOTMETH_EXTLINUX_PXE`
      which are what we want
    - it selects `CONFIG_BOOT_DEFAULTS` which selects
      `CONFIG_BOOT_DEFAULTS_CMDS` which implies `CONFIG_USE_BOOTCOMMAND`
    - as a result, `CONFIG_BOOTCOMMAND` defaults to `bootflow scan`
  - `CONFIG_BOOTSTD_FULL` enables additional cmds
  - `CONFIG_BOOTSTD_BOOTCOMMAND` affects the default bootcmd
  - `CONFIG_BOOTSTD_PROG` causes `main_loop` to call `bootstd_prog_boot`
    directly to boot
- autoboot
  - `CONFIG_AUTOBOOT` is enabled by default
    - `main_loop` calls `autoboot_command` with the string from `bootcmd` env
    - the default value of `bootcmd` env is `bootflow scan`
  - `U_BOOT_CMD_WITH_SUBCMDS(bootflow, ...)` defines the `bootflow` command
    - `do_bootflow_scan` calls `bootflow_scan_first` and `bootflow_scan_next`
      to loop over all bootflows
      - it calls `bootdev_add_bootflow`
      - it calls `bootflow_run_boot` to boot each bootflow in turn
        - this calls `bootmeth_boot`
- `bootflow_scan_first`
  - `bootmeth_setup_iter_order` collects all `UCLASS_BOOTMETH`
  - `bootdev_setup_iter` finds the first bootdev
    - it seems to support `UCLASS_BOOTSTD`, bootdev hunter, and
      `UCLASS_BOOTDEV`
    - if `boot_targets` env is set, `bootstd_get_bootdev_order` returns the
      env and `bootdev_next_label` tries them in order
    - `bootdev_next_prio` finds and probes `UCLASS_BOOTDEV`
    - `dm_scan_other` adds a default `UCLASS_BOOTSTD` device and a
      `UCLASS_BOOTMETH` device for each `UCLASS_BOOTMETH` driver
    - mmc registers a boodev hunter with `BOOTDEV_HUNTER(mmc_bootdev_hunter)`
  - `bootflow_check` calls `bootmeth_get_bootflow`
- `bootflow_scan_next` finds the first partition of the current device, or the
  first partition of the next device, or fail
- `bootmeth_get_bootflow` calls `read_bootflow` callback
  - `extlinux_read_bootflow` reads `/extlinux/extlinux.conf` or
    `/boot/extlinux/extlinux.conf` on the partition
    - the paths are from `default_prefixes` and `EXTLINUX_FNAME`
  - `efi_mgr_read_bootflow` finds `BootOrder` from efi vars
    - `efi_init_obj_list` calls `efi_init_variables` calls `efi_var_from_file`
      - it reads `EFI_VAR_FILE_NAME` (`ubootefi.var`)
  - `extlinux_pxe_read_bootflow` calls `pxe_get` to retrieve the boot file
    - it uses `do_get_tftp` which waits for the network
    - `PXELINUX_DIR` is `pxelinux.cfg/`
    - `pxe_default_paths` has various default filenames
- `bootmeth_boot` calls `boot` callback
  - `extlinux_boot` calls `pxe_process` to parse and boot the entry
    - the format of `extlinux.conf` appears to be based on pxelinux, extlinux,
      and
      <https://uapi-group.org/specifications/specs/boot_loader_specification/>
  - `efi_mgr_boot` calls `efi_bootmgr_run`
  - `extlinux_pxe_boot` also calls `pxe_process`

## Usage

- Environment Variable Commands
  - `printenv`
  - `setenv bootargs 'console=ttyS0,115200'` sets kernel cmdline
    - or `console=tty0` or both
  - `saveenv`
- Storage Commands
  - `usb reset` rescans USB devices
  - `usb storage` lists USB storage devices
  - `usb part` lists USB storage partitions
  - `ls usb 0:1` lists files in USB storage device 0 partition 1
  - `load usb 0:1 0x3000000 vmlinuz` loads kernel to 0x3000000
  - `load usb 0:1 0x6000000 initramfs.img` loads initramfs to 0x6000000
- Boot Commands
  - `zboot 0x3000000 - 0x6000000 ${filesize}` boots bzImage at 0x3000000 with
    initramfs at 0x6000000
    - note that initramfs size is required and is in hex
    - `Valid Boot Flag`
    - `Setup Size = 0x00003e00`
    - `Magic signature found`
    - `Using boot protocol version 2.0f`
    - `Linux kernel version ...`
    - `Building boot_params at 0x00090000`
    - `Loading bzImage at address 100000 (9398496 bytes)`
    - `Magic signature found`
    - `Kernel command line: "console=ttyS0,115200"`
    - `Magic signature found`
    - `Starting kernel ...`
    - after the kernel initializes the console, it prints the banner
      - `[    0.000000] Linux version ...`
  - `boot` runs the commands in `bootcmd`
- Automatic Boot
  - `setenv bootdelay 5`
  - `setenv bootcmd '...'` for semicolon-separated commands
  - `setenv bootargs '...'` for kernel cmdline
  - it will run `boot` after the delay
- On my testing machine,
  - u-boot is really slow because of fb init and updates
  - there is no `saveenv`
  - mmc is not supported; has to use usb storage
  - loading kernel/initramfs to 0x1000000/0x2000000 does not work
    - initramfs seems to be corrupted
  - `console=ttyS0,115200` stops working after switching from Debian kernel to
    Arch kernel
    - probably just a loglevel issue?
  - login takes ~20 seconds for some unknown reason

## Old

- env

    #define CFG_NO_FLASH            1
    #define CFG_ENV_IS_IN_NAND      1
    #define CFG_ENV_SIZE            0x40000 /* 128k Total Size of Environment Sector
    #define CFG_ENV_OFFSET_OOB      1       /* Location of ENV stored in block 0 OOB
    #define CFG_PREBOOT_OVERRIDE    1       /* allow preboot from memory */
    #define NAND_MAX_CHIPS          1
    #define CFG_NAND_BASE           0x4e000000
    #define CFG_MAX_NAND_DEVICE     1
    
    CFG_ENV_OFFSET is defined to be env_offset, which is set to CFG_OFFSET_OOB

- all starts at `lib_arm:start_armboot`
- `env_init` set directly `gd->env_addr` to `default_environment` and `gd->env_valid = 1`
- after `nand_init`, `env_relocate` is called, which

    env_ptr is set to malloc(some size);
	env_get_char = env_get_char_memory;
	if (preboot_override) gd->env_valid = 0;
	if (gd->env_valid) env_relocate_spec() else default_env();
	gd->env_addr = env_ptr->data;
- `Found Environment offset in OOB..` -> try to read env from nand
  - if not `*** Warning - bad CRC or NAND, using default environment\n\n` -> env from nand is used
`main_loop`

    if preboot_override, run_command(p);
    bootdelay set, s = bootcmd, then
    if (!nobootdelay && bootdelay >= 0 && s && !abortboot (bootdelay)) run_command(s);
    	abortboot return true when any key pressed
- `nand` command:
  - many nand subcommands indirectly calls `mtdparts_init()`, which gets parts
    info from `getenv("mtdparts")` (and others)
- `dynenv` command:
  - it indirectly calls into `mtdparts_init` too.  On return, `CFG_ENV_OFFSET` is
    set to the new address
- `saveenv` save env to `CFG_ENV_OFFSET`
