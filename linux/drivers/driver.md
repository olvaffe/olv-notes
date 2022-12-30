Kernel Driver 123
=================

## `/sysfs/bus`

- `buses_init`
  - `bus_kset` is created with `kset_create_and_add` with no parent kobject
  - `system_kset` is created with `kset_create_and_add` under `devices_kset`
- `bus_register` sets `bus->p` to a newly allocated `subsys_private`
  - `priv->subsys` is a kset under `bus_kset`
  - `priv->devices_kset` is a kset under `priv->subsys`
  - `priv->drivers_kset` is a kset under `priv->subsys`

## `/sysfs/class`

- `classes_init`
  - `class_kset` is created with `kset_create_and_add` with no parent kobject
- `class_register` sets `cls->p` to a newly allocated `subsys_private`
  - `priv->subsys` is a kset under `class_kset`
  - unlike `bus_register`, no `priv->devices_kset` nor `priv->drivers_kset`

## `/sys/devices`

- `devices_init`
  - `devices_kset` is created with `kset_create_and_add` with no parent
    kobject
  - `dev_kobj` is created with `kobject_create_and_add` with no parent kobject
  - `sysfs_dev_block_kobj` is created with `kobject_create_and_add` under
    `dev_kobj`
  - `sysfs_dev_char_kobj` is created with `kobject_create_and_add` under
    `dev_kobj`
- `device_register`
  - `dev->kobj.kset` is initialized to `devices_kset`
  - for a non-class device, its parent is
    - the explicitly specified parent, if specified,
    - the default root (`dev_root`) of the bus, if exists,
    - NULL
  - for a class device, its parent is
    - `virtual_device_parent`, if none specified
    - the explicitly specified parent, if the parent has a class
    - a glue dir
- many of the top-level nodes are PMUs registered by `perf_pmu_register`
  - `breakpoint` from `init_hw_breakpoint`
  - `cpu` from `init_hw_perf_events`
  - `cstate_core` and `cstate_pkg` from `cstate_init`
  - `i915` from `i915_pmu_register`
  - `intel_pt` from `pt_init`
  - `msr` from `msr_init`
  - `power` from `rapl_pmu_init`
  - `software` from `perf_event_init`
  - `tracepoint`, `kprobe`, and `uprobe` from `perf_tp_register`
  - `uncore_arb`, `uncore_cbox_0`, `uncore_cbox_1`, and `uncore_imc` from
    `uncore_pmu_register`
- `isa` is by `isa_bus_init`
- `LNXSYSTM:00` is by `acpi_device_add` on `ACPI_SYSTEM_HID`
- `pci0000:00` is by `pci_register_host_bridge`
- `platform` is by `platform_bus_init`
- `pnp0` is by `pnp_register_protocol`
- `wakeup0` is by `pm_autosleep_init`
- `system` is a kset by `buses_init`
  - `cpu` is from `cpu_dev_init`
  - `memory` is from `memory_dev_init`
  - don't use `subsys_system_register` in new code
- `virtual` is by `virtual_device_parent`
  - `workqueue` is from `wq_sysfs_init`, a virtual subsys
  - `dma_heap` is from `dma_heap_add`, a class device with no parent

## `/sys/module`

- `param_sysfs_init`
  - `module_kset` is created with `kset_create_and_add`
  - `version_sysfs_builtin` adds built-in modules and their `version` for all
    built-in modules with `MODULE_VERSION`
  - `param_sysfs_builtin` adds built-in modules and their params for all
    built-in modules with `module_param_named`
    - `module_param_named` can specify to hide a param though
  - a built-in module without `MODULE_VERSION` or without visible params does
    not show up in `/sys/module`
- `init_module` syscall calls `load_module`
  - in `mod_sysfs_init` called from `mod_sysfs_setup`, the module kobj is
    added with `module_kset` as the kset
- when a module registers a driver, `module_add_driver` calls
  `module_create_drivers_dir` to add `/sys/module/<MODULE>/drivers`
  - unless it is a built-in module that does not set
    `device_driver::mod_name`, which is very common unfortunately

## Drivers

- all `device_driver` can be found under `/sys/bus/*/drivers`
- driver core adds a cpu bus
  - `processor` adds CPU idle/freq/hotplug/thermal/throttle supports
- ACPI is parsed and a `acpi_device` tree is constructed.  Only a handle of
  `acpi_device` is driven directly
  - `ac` registers a `power_supply` and detects if the system is on AC or
    battery 
  - `battery` registers `power_supply`s and shows battery status
  - `button` registers `input_dev`s and reports power/sleep/lid keys
  - `ec` is a low-level detail of ACPI?
  - `processor_aggregator` detects `ACPI_PROCESSOR_AGGREGATOR_NOTIFY` which is
    ACPI requesting OS to idle a CPU.  The driver context switches the CPU to
    `power_saving_thread`
  - `thermal` registers `thermal_zone_device`s and shows the thermal status
  - `tpm_crb` registers a `tpm_chip`
  - `video` registers `input_dev`s (and sometimes `backlight_device`s) and
    reports brightness keys
- pci bus
  - `ahci` registers a `Scsi_Host` for each port in the ATA controller
  - `bdw_uncore` driver the CPU uncore as a pci device on the bus
  - `ehci-pci` drives the EHCI USB controller and registers a `usb_hcd`
  - `i801_smbus` drives the SMBus controller and registers a `i2c_adapter`
  - `i915` drives the Intel GPU
  - `intel_pch_thermal` drives the Intel PCH and registers a
    `thermal_zone_device`
  - `xhci_hcd` drives the xHCI USB controller and registers a `usb_hcd`
  - `lpc_ich` drives Intel LPC controller, where legacy devices such as
    `iTCO_wdt` or `intel-spi` are connected to
    to
  - `mei_me` drives Intel MEI controller and registers a mei bus
  - `pcieport` drives pcie root ports and registers more pci buses
  - `proc_thermal` drivers Intel Thermal controller
  - `rtsx_pci` drives realtek pcie card reader and registers a
    `rtsx_pci_sdmmc` platform device
  - `snd_hda_intel` drives intel audio controllers and calls
    `azx_probe_codecs` to register `hda_codec`s.
- `pci_express` bus
  - pcie root ports are conventional PCI devices
  - service and `pcie_pme`
- usb bus
  - USB controllers are driven by `ehci-pci`
  - `usb` is a `usb_device_driver` that is bound to usb devices (not interfaces)
    - this is almost the only `usb_device_driver`
  - `hub` is a `usb_driver` that is bound to hub interfaces
  - `btusb` is a `usb_driver`
- scsi bus
  - a sata controller adds a scsi bus for each port
  - a disk is a scsi target on the bus
  - `sd` registers `gendisk` for a disk scsi target
- pnp bus
  - pnp acpi adds pnp devices on the bus
  - `system` reserves resources for some devices
  - `rtc_cmos` registers a `rtc_device` and a `nvmem_device`
  - `i8042 kbd` and `i8042 aux` helps `i8042` registers serio devices, which
    are driven by `atkbd` and `psmouse`
  - `serial` registers a 8250 port which is driven by `serial8250`
- i2c bus
  - `i2c_hid` drivers touchpad
- hid subsys adds hid bus
  - `hid-multitouch` drives touchpad
- hda subsys adds hdaudio bus
  - `snd_hda_codec_hdmi` drivers HDMI audio
  - `snd_hda_codec_realtek` drivers audio card
- mei bus
  - intel ME driver adds a mei bus
  - `mei_hdcp` is a mei client driver
- nd bus
  - no device
- serio bus
  - acpi describes two i8042 ports, which the driver adds to serio bus
  - `atkbd` drives kbd port, if a keyboard is connected
  - `psmouse` drives aux port, if a mouse is connected
- spi bus
  - Intel ICH has a `intel-spi` spi controller connected to flash to expose
    the flash (for BIOS update)
  - `spi-nor` is used by `intel-spi`
- platform bus
  - `acpi-fan` registers a thermal cooling device
  - `acpi-wmi` register a wmi bus
  - `alarmtimer` 
  - `clk-lpt` registers a clk for Intel LPSS
  - `coretemp` registers a hwmon device when CPU is hotplugged
  - `dw_dmac` registers a DMA engine for Intel LPSS
  - `efi-framebuffer` registers a fbdev
  - `i2c_designware` registers i2c adapter
  - `i8042`
  - `int3400 thermal` registers a `thermal_zone_device`
  - `intel_rapl_msr` registers a `powercap_control_type`
  - `intel-spi` registers a spi bus and a mtd device via `spi-nor`
  - `iTCO_wdt` registers a `watchdog_device` device
  - `PCCT` registers `mbox_controller`
  - `pcspkr` registers `input_dev` to take `EV_SND` from vt or userspace
  - `reg-dummy` registers a dummy `regulator_dev`
  - `serial8250`
  - `rtsx_pci_sdmmc` registers a `mmc_host`

## Device Model

- `/sys/devices` is hierarchical.
- The others might not.  The goal of other directories is not to give a
  hierarchical view of the device tree.
  - E.g. `/sys/bus/pci/devices` contains all pci devices in a flat view.
- A SDHCI host controller lives on the PCI bus and has class mmc_host.  The card
  inserted lives on mmc bus, and is a child of the mmc_host?
- `struct device` has
  - `struct device *parent`, the device's parent
  - `struct device_private *p`, for deivce core use, like the device's children
  - `struct device_type *type`, the type of the device
  - `struct bus_type *bus`, the type of bus the device is on
  - `struce device_driver *driver`, the driver of the device
  - `struct class *class`, the class of the device
- A device with `class` is called a class device.  When a class device is
  added, it is put under a subdirectory with its class name of its parent,
  unless its parent is also a class device.
  - E.g. `pcspkr` has no class and `platform` is its parent.  It will be
    found under `/devices/platform/pcspkr`.
  - E.g. `hwmon0` is a hwmon class device and it has no parent.
    It will be found under `/devices/virtual/hwmon/hwmon0`.
  - E.g. `hwmon1` is a hwmon class device and `coretemp.0` is its parent.  It
    will be found under `/devices/platform/coretemp.0/hwmon/hwmon1`.
- Where a device is put is decided by `get_device_parent`.
  - It decides the device's kobject parent.
  - the device's `class` and `parent` can be null.
  - if both of them is null, the device has no kobject parent.
  - if `class` is null and `parent` is non-null, the parent's kobject is the
    device's kobject parent.
  - if `class` is non-null, the kobject parent is
    - if the device has a `parent` and the `parent` has a `class`, then the
      parent's kobject is the device's kobject parent.
    - if there is no `parent`, `virtual` is used as a parent without `class`.
    - if the `parent` has no `class`, a new kobject is created and added as the
      parent's child and as the device's kobject parent.
- It should be noted that what is in the sysfs are decided by `kset` and
  `kobject`.
  - The hierarchy is not decided by device and its parent.  Otherwise, a device
    without class or parent would be found under `/`, not `/devices`.
  - A `kobject` is a directory holding files.
  - A `kset` is a `kobject` that can also hold `kobject`s.
- There are `struct device platform_bus` and `struct bus_type platform_bus_type`.
  - `platform_bus` has no class, nor parent, nor bus.  It can be found under
    `/devices/platform`.
  - `platform_bus_type` is not a device.  It has a `subsys` kobject whose kset
    is `bus_kset`, and this makes it accessible under `/bus/platform`.  It has
    two ksets under `subsys`, which correspond to `/bus/platform/devices` and
    `/bus/platform/drivers`.
  - Bus is used for device-driver matchings.
- Take a look at SDHCI controller attached to the PCI bus.
  - There is a PCI device `/sys/devices/pci0000:00/0000:00:1e.0/0000:03:0b.3`.
  - When its driver, `sdhci_pci`, is attached, it creates a mmc_host class
    device whose parent is the pci device
    `/devices/pci0000:00/0000:00:1e.0/0000:03:0b.3/mmc_host/mmc0`
  - When an SD card is inserted, the card is a device with parent mmc0
    `/devices/pci0000:00/0000:00:1e.0/0000:03:0b.3/mmc_host/mmc0/mmc0:e624`.
    - It lives on the `mmc` bus `/bus/mmc/devices/mmc0:e624`.
- It seems that class devices are usually functions of a physical device.
  - Those not under this category can be found under `/devices/virtual`.

## Random Notes

- open fileoperation has prototype `int (*open) (struct inode *, struct file *)`
  `struct inode` is the file on the fs, while `struct file` is a fd
- char node major number is a limited resouce.  `register_chrdev_region` to
  reserve a range.

## Memory Access

- CPU/system/real memory could be expressed in:
  - physical address: physical
  - virtual address: the way cpu sees it
  - bus address: the way a bus sees it
- Physical and bus addresses coincide on x86.
- PCI/io/shared memory is usually not DRAM (hardware-wise).  It is _NOT_ in the
  same address space as system memory is.  It should be accessed through
  `readb/writeb`.
  - It is in the same address space as system memory is on x86. 
- ISA/LPC DMA
 - `Documentation/DMA-ISA-LPC`
 - depend on `ISA_DMA_API`
 - `GFP_DMA` is for ISA DMA.
 - do not use `isa_virt_to_phys` because it depends on ISA

## Locking

- `local_irq_save` to avoid race with current cpu
- `spin_lock` to avoid race with another cpu
- Documentation/spinlocks.txt
- Concurrency: Two or more functions could be called concurrently.
- Reentrancy: The same function could be concurrently called.
- Mutex (kernel/mutex.c)
  - sleep with `__set_task_state(task, TASK_UNINTERRUPTIBLE); schedule();`
  - needs to be waked up

## TLB

- cachetlb.txt is a good read

## Module

- device tables are extracted from .o by `scripts/mod/file2alias.c`.  They are
  parsed and aliases are added to the final `.ko`.
