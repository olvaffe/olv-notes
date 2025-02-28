Kernel irqchip
==============

## ARM Generic Interrupt Controller, version 3

- GICv3 provides private peripheral interrupts (PPI), shared peripheral
  interrupts (SPI), software generated interrupts (SGI), and locality-specific
  peripheral interrupts (LPI)
- DT
  - `compatible = "arm,gic-v3";` identifies the driver
  - `interrupt-controller;` identifies the node as an interrupt controller
  - `#interrupt-cells` is at least 3
    - 1st cell is the interrupt type: 0 for SPI, 1 for PPI
    - 2nd cell is the interrupt number: 0-998 for SPI, 0-15 for PPI
    - 3rd cell is the flags: edge, level, etc.
    - 4th cell is for PPI: a node in `ppi-partitions` for cpu affinity
  - `interrupts` specifies the maintenance irq
    - it triggers when a guest changes the state of its vgic
- `IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init);`
  - this adds the compat string to `__irqchip_of_table`
  - when `start_kernel` calls `init_IRQ`, arm64 calls `irqchip_init`
    - `of_irq_init` finds the matching nodes and calls the init func
  - `gic_of_init`
    - `gic_init_bases` calls `irq_domain_create_linear` to create an
      `irq_domain`
