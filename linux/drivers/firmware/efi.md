Kernel EFI
==========

## x86

- `CONFIG_EFI_STUB`
  - `arch/x86/boot/header.S` constructs PE header for efi
  - `efi_pe_entry` is the entry point
    - `efi_allocate_bootparams` allocs boot params
      - `efi_convert_cmdline` copies cmdline
    - `efi_decompress_kernel` decompresses bzimage
    - `efi_load_initrd` loads initrd
    - `setup_graphics` sets up graphics
      - it calls `efi_setup_graphics` to query efifb info and init
        `boot_params->screen_info` and `boot_params->edid_info`
    - `enter_kernel` jumps to start of the decompressed kernel
- x86 `setup_arch` initializes efi
  - `EFI_BOOT` and `EFI_64BIT` bits are set in `efi.flags`
  - `efi_init`
    - `efi_systab_init` parses the efi header and initializes things such as
      `efi_runtime`, `efi.runtime_version`, etc.
    - `EFI_RUNTIME_SERVICES` bit is set
- x86 `efi_enter_virtual_mode` is called
  - `efi.runtime` is set to `efi_runtime`
  - `__efi_enter_virtual_mode` sets up the address space for use by efi
    runtime services
  - `efi_native_runtime_setup` initializes `efi` callbacks
- efi callbacks, using `efi.get_wakeup_time` as an example
  - the callback points to `virt_efi_get_wakeup_time`
  - `virt_efi_get_wakeup_time` calls `efi_queue_work(GET_WAKEUP_TIME)`
  - `efi_call_rts` calls `efi_call_virt(get_wakeup_time)`
    - this calls `__efi_call` defined in `efi_stub_64.S`
      - it translates from sysv calling convention to uefi calling convention
      - it then calls the address in the first argument
    - the first argument is passed in `%rdi` and contains
      `efi.runtime->get_wakeup_time`, where `efi.runtime` points to an address
      in efi and is controlled by efi

## earlycon

- this is helpful to debug efi boot issue without serial port or working
  display driver
- `debug` logs up to `KERN_DEBUG` messages to console
  - by default, it logs up to `KERN_INFO` to console
- `boot_delay=200` sleeps 200ms between each message
- `earlycon=efifb` uses efifb as the console in early boot
  - the screen stops updating after
    - `Console: colour dummy device 80x25`
    - `printk: legacy console [tty0] enabled`
    - `printk: legacy bootconsole [efifb0] disabled`
  - this is because vt has replaced efifb as the console, but vt is using
    `dummy_con` and does not update the screen yet
  - `keep_bootcon` keeps efifb as the console
    - printk will log to both efifb and vt
    - it will be problematic when vt switches to `fb_con` and conflicts with
      efifb
    - only useful when vt `fb_con` does not work
- vt `fb_con` logging
  - `fb_con` replaces `dummy_con`, clears the screen, and starts logging after
    - `[drm] Initialized simpledrm 1.0.0 for simple-framebuffer.0 on minor 0`
      - or whatever built-in drm driver or legacy fbdev driver
    - `Console: switching to colour frame buffer device 128x48`
      - this says `fb_con` replaces `dummy_con`
- the final display driver probes
  - `aperture_remove_conflicting_pci_devices` removes the simple display
    driver and vt switches back to `dummy_con`
    - `Console: switching to colour dummy device 80x25`
  - `register_framebuffer` registers a new fb to `fb_con` and vt switches to
    `fb_con`
    - `Console: switching to colour frame buffer device 160x45`
- if the final display driver (or its deps) misbehaves, the screen can turn
  black
  - on arm, this can happen with display, bridge, panel, clk, regulator,
    pinctrl, pmdomain, etc.
