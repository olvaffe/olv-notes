Kernel Driver
=============

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

