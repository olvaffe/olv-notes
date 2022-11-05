Das U-Boot
==========

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
