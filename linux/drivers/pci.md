PCI
===

## Specs

- PCIe 1.0
  - 2003
  - 2.5 GT/s
  - 8b/10b (encode 8-bit words to 10-bit symbols)
  - x1: 2.5 GT/s * 8 / 10 = 2 Gb/s = 0.25 GB/s
  - x16: 4 GB/s
- PCIe 1.1
  - 2005
  - 2.5 GT/s
- PCIe 2.0
  - 2007
  - 5 GT/s
  - 8b/10b 
  - x1: 0.5 GB/s
  - x16: 8 GB/s
- PCIe 2.1
  - 2009
  - 5 GT/s
- PCIe 3.0
  - 2010
  - 8 GT/s
  - 128b/130b
  - x1: 8 GTs * 128 / 130 = 7.877 Gb/s = 0.985 GB/s
  - x16: 15.754 GB/s
- PCIe 3.1
  - 2014
  - 8 GT/s
- PCIe 4.0
  - 2017
  - 16 GT/s
  - x1: 1.969 GB/s
  - x16: 31.508 GB/s
- PCIe 5.0
  - 2019
  - 32 GT/s
  - x1: 3.938 GB/s
  - x16: 63.015 GB/s
- PCIe 6.0
  - 2021
  - 64 GT/s
  - x1: 7.877 GB/s
  - x16: 126.031 GB/s

## PCI Architecture

- there is a host bus
- CPUs, L3, and memory controller are connected to the host bus
- PCI Host Bridge is also connected to the host bus
  - it bridges the host bus with the PCI bus
- PCIe

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
    - the root bus ops is `acpi_pci_root_ops` and `pci_root_ops`
    - `pci_bus_add_devices`
- `arch_init_msi_domain` calls `pci_msi_create_irq_domain` with
  `pci_msi_domain_info` to create "PCI-MSI" irq domain
- `pcibios_*` are defined by arch

## PCI Config Space

- `pci_read_config_*`
  - `pci_bus_read_config_*`
  - `pci_read`
  - `raw_pci_read`
  - `pci_conf1_read`

## PCI Interrupt

- either MSI/MSIX or one of the four interrupt pins are used
  - `dev->pin` is one of the four PCI interrupt pins
  - two PCI devices can share the same pin
  - always use MSI/MSIX
- when MSI is not enabled for a device,
  - `pcibios_enable_device` is called by `pci_enable_device`
  - `pcibios_enable_irq` points to `acpi_pci_irq_enable`
  - `acpi_pci_irq_lookup` finds the GSI for the interrupt pin
  - `acpi_register_gsi` calls `acpi_register_gsi_ioapic` to find the kernel
    irq number for the GSI
    - `mp_find_ioapic` finds the ioapic for the gsi
    - `mp_find_ioapic_pin` finds the pin (hwirq)
    - `mp_map_pin_to_irq` finds the kernel virq number from the pin
      - or calls `__irq_domain_alloc_irqs` to set it up
- MSI can be enabled by `pci_alloc_irq_vectors` with `PCI_IRQ_MSI*`
  - `pci_enable_msi` or `pci_enable_msix_*` are deprecated
  - `msi_capability_init` enables MSI and sets `dev->irq`
    - `msi_setup_entry` allocs a `msi_desc`
    - `pci_msi_setup_msi_irqs` calls `arch_setup_msi_irqs` which calls
      `native_setup_msi_irqs`
    - `msi_domain_alloc_irqs`
      - `pci_msi_prepare`
      - `pci_msi_set_desc` sets `msi_hwirq`
      - `__irq_domain_alloc_irqs` allocates irqs
        - `irq_domain_alloc_descs` allocs `irq_desc`
        - `irq_domain_alloc_irqs_hierarchy` recursively allocs from the irq
          domains
          - `x86_vector_alloc_irqs`
            - `reserve_irq_vector` reserves the irq
          - `intel_irq_remapping_alloc`
          - `msi_domain_alloc`
            - `pci_msi_get_hwirq`
            - `msi_domain_ops_init` sets `irq_data` and handler
              - the domain info is `pci_msi_domain_info`
      - `irq_set_msi_desc_off` updates `msi_desc`
      - `irq_domain_activate_irq` activates the irq
        - it recursively activates the irq domains from the root
	- `x86_vector_activate` updates `vector_irq` that `do_IRQ` uses to
	  look up irq desc
	- `intel_irq_remapping_activate` updates vt-d irte (Interrupt
	  Remapping Table Entry)
        - `msi_domain_activate` calls `pci_msi_domain_write_msg`
  - `pci_intx_for_msi` disables interrupt pin
  - `pci_msi_set_enable` enables MSI
- `request_irq`
  - `__setup_irq` is the gut
  - `irq_activate` activates the irq
    - no-op since it has been activated when enabling MSI
  - `irq_startup` unmasks the irq
    - `irq_enable`
    - `unmask_irq` and `pci_msi_unmask_irq`
    - `irq_setup_affinity`
      - `msi_domain_set_affinity`
      - `intel_ir_set_affinity`
      - `apic_set_affinity`

## PCI driver

- `pci_driver` and `pci_register_driver`
