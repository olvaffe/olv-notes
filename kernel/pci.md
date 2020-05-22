PCI
===

## PCI Setup

- `pci_setup` parses pci= cmdline
  - on x86, `pci_probe` defaults to
    - `PCI_PROBE_BIOS`
    - `PCI_PROBE_CONF1`
    - `PCI_PROBE_CONF2`
    - `PCI_PROBE_MMCONF`
- `acpi_pci_init` is an arch initcall
  - it registers `acpi_pci_bus`
- `pci_arch_init` is an arch initcall
  - `pci_direct_probe`
    - sets `raw_pci_ops` to `&pci_direct_conf1`
  - `pci_mmcfg_early_init`
    - `pci_parse_mcfg` parses ACPI MCFG table
      - `/sys/firmware/acpi/tables/MCFG`
      - `struct acpi_mcfg_allocation` example
        - `address = 0xf8000000`
        - `pci_segment = 0`
        - `start_bus_number = 0`
        - `end_bus_number = 0x3f`
      - `pci_mmconfig_add` adds a new `pci_mmcfg_region` to `pci_mmcfg_list`
    - `__pci_mmcfg_init`
      - `pci_mmcfg_reject_broken` checks if the mmconfig is reserved in E820
      - `pci_mmcfg_arch_init` ioremaps the region and sets `raw_pci_ext_ops`
      	to `&pci_mmcfg`
- `acpi_init` is a subsys initcall
  - `acpi_pci_root_init` is called from `acpi_scan_init`.  It adds
    `pci_root_handler` as a scan handler.
  - PCI root device has `_HID` PNP0A08 and `_CID` PNP0A03
  - `acpi_pci_root_add` adds the PCI root bridge
    - `pci_acpi_scan_root` calls `acpi_pci_root_create` which calls
      `pci_create_root_bus` to create and register a `pci_bus`
      - `pci_scan_child_bus` and `pci_scan_slot` scans all slots
    - `pci_bus_add_devices`

## PCI driver

- `pci_enable_msi`
- `pci_enable_device`
  - `do_pci_enable_device`
  - `pcibios_enable_device`
    - `pcibios_enable_irq` points to `acpi_pci_irq_enable`
      - not used if msi enabled
- `pci_driver` and `pci_register_driver`
