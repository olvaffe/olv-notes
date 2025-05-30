Device Tree
===========

## Devicetree Specification

- <https://www.devicetree.org/specifications>
- Chapter 1 Introduction
  - 1.1 Purpose and Scope
  - 1.2 Relationship to IEEE™ 1275 and ePAPR
  - 1.3 32-bit and 64-bit Support
  - 1.4 Definition of Terms
    - `cell` A unit of information consisting of 32 bits
- Chapter 2 The Devicetree
  - 2.1 Overview
  - 2.2 Devicetree Structure and Conventions
    - a node name follows `node-name@unit-address` convention
      - if the node has no `reg`, `@unit-address` must be omitted
      - `node-name` should be generic, such as `bus`, `clock`, `cpu`, `disk`,
        `dsp`, `efuse`, `gpio`, `gpu`, `iommu,` `mailbox`, etc.
    - a node has properties that consist of a name and a value
      - non-standard property names should be namespaced, such as
        `linux,network-index`
      - a property value is an array of zero or more bytes
        - `<empty>` is empty, used when the prop is bool and the presense or
          absense of the prop suffices
        - `<u32>` is 32-bit int
        - `<u64>` is 64-bit int
        - `<string>` is a null-terminated string
        - `<prop-encoded-array>` is opaque
        - `<phandle>` is a `<u32>` referencing another node
        - `<stringlist>` is a list of `<string>`
  - 2.3 Standard Properties
    - `compatible: <stringlist>`
    - `model: <string>`
    - `phandle: <u32>` is a unquie id for the node
      - it is automatically generated by tools
    - `status: <string>`
      - `okay` is the default, if the prop is missing
      - `disabled` means non-operational for now
      - `fail` means non-operational forever
    - `#address-cells: <u32>` and `#size-cells: <u32>`
      - if the node has children with `reg`, they indicate the number of cells
        for children `reg`
      - for 64-bit address space, both are 2
    - `reg: <prop-encoded-array>`
      - `(addr, size)` pairs, usually for mmio regions
    - `virtual-reg: <u32>`
    - `ranges: <prop-encoded-array>` and `dma-ranges: <prop-encoded-array>`
      - `(child-bus-address, parent-busaddress, length)` triplets
    - `dma-coherent: <empty>` indicates coherent device on systems without
      system-level coherency
    - `dma-noncoherent: <empty>` indicates non-coherent device on systems with
      system-level coherency
  - 2.4 Interrupts and Interrupt Mapping
    - `interrupts: <prop-encoded-array>` indicates an interupt-generating
      device
    - `interrupt-parent: <phandle>` indicates an interrupt parent when differs
      from the node parent
    - `interrupts-extended: <phandle> <prop-encoded-array>...` conflicts with
      `interrupts` and is more flexible
    - `#interrupt-cells` indicates the cell count to encode an interrupt
    - `interrupt-controller` indicates an interrupt controller
  - 2.5 Nexus Nodes and Specifier Mapping
- Chapter 3 Device Node Requirements
  - 3.1 Base Device Node Types
    - there should be a root node, a `/cpus` node, and one or more `/memory`
      nodes
  - 3.2 Root node
  - 3.3 `/aliases` node
    - each property defines an alias
      - `serial0 = "/simple-bus@fe000000/serial@llc500";`
    - dtc uses labels to reference nodes during compile time
    - kernel uses aliases to reference nodes during runtime
  - 3.4 `/memory` node
  - 3.5 `/reserved-memory` Node
  - 3.6 `/chosen` Node
    - this node is often generated by the bootloader
    - `bootargs: <string>` indicates the kernel cmdline
  - 3.7 `/cpus` Node Properties
    - this is a container for `cpu` nodes
  - 3.8 `/cpus/cpu*` Node Properties
    - `reg` indicates the cpu ids
    - `enable-method: <stringlist>` indicates how to bring up the cpu
  - 3.9 Multi-level and Shared Cache Nodes (`/cpus/cpu*/l?-cache`)
- Chapter 4 Device Bindings
  - the requirements, aka bindings, on how a specific type/class of devices
    are represented in DT should be defined
  - 4.1 Binding Guidelines
    - `clock-frequency: <prop-encoded-array>` indicates clock freq in hz
    - `reg-shift: <u32>` indicates the width of registers
    - `label: <string>` is a human-readable description of the device
  - 4.2 Serial devices
    - `current-speed: <u32>` specifies the baud rate
  - 4.3 Network devices
  - 4.4 Power ISA Open PIC Interrupt Controllers
  - 4.5 simple-bus Compatible Value
- Chapter 5 Flattened Devicetree (DTB) Format
  - the filename should end with `.dtb`
    - i guess dtb is the exchange format and is parsed into fdt in-memory
  - 5.1 Versioning
  - 5.2 Header
    - `struct fdt_header`
    - magic, total size, version, offsets/sizes of struct/string/mem-resv
      blocks, etc.
  - 5.3 Memory Reservation Block
    - a list of `fdt_reserve_entry` to delimit reserved regions
  - 5.4 Structure Block
    - represents the dt as a linear tree
    - `FDT_BEGIN_NODE` marks the begin of a node, followed by the node name
      - can be nested to form a tree
    - `FDT_PROP` marks a node prop, followed by the prop name and value
    - `FDT_END_NODE` marks the end of a node
    - `FDT_NOP` should be ignored and is used to patch things out
    - `FDT_END` marks the end of structure block
  - 5.5 Strings Block
    - contains all used property names
  - 5.6 Alignment
- Chapter 6 Devicetree Source (DTS) Format (version 1)
  - the filename should end with `.dts`
  - 6.1 Compiler directives
    - `/include/ "foo.dtsi"` includes another file
      - the filename should end with `.dtsi`
  - 6.2 Labels
    - labels are used in place of explicit phandle values or absolute node
      paths
      - they are not encoded into dtb
    - `foo: bar {...}` creates label `foo` for node `bar`
    - `&foo` references label `foo`
  - 6.3 Node and property definitions
    - `[label:] node-name[@unit-address] { [properties definitions] [child nodes] };`
      defines a node
      - `/delete-node/ node-name;` or `/delete-node/ &label;` deletes a
        previously-defined node
    - `[label:] property-name = value;` or `[label:] property-name;` defines a
      property
      - `/delete-property/ property-name;` deletes a previously-defined
        property
    - property values
      - array of 32-bit integer cells: `<13 0x1a>`
      - `<u64>` is represented by 2 cells: `<0x00000001 0x00000000>`
      - string: `"foo"`
      - bytestring: `[00 11 22 aa bb cc]`
      - comma-separated list: `"foo", "bar"`
      - phandle reference: `&label` or `&{/full/path/to/node}`
  - 6.4 File layout
    - `/dts-v1/;` indicates dts v1
    - zero or more `/memreserve/ <address> <length>;`
    - `/ { ... };` for root node
    - c and c++ style comments are supported

## Initialization

- during init, `unflatten_device_tree` is called to unflatten fdt passed in by
  the firmware
  - `__unflatten_device_tree` is called and `of_root` is set
    - `unflatten_dt_alloc` returns a `device_node`
    - `of_node_init` calls `fwnode_init`
  - `of_alias_scan` parses `/chosen` and `/aliases`
    - `of_chosen` points to `/chosen`
      - `of_stdout` points to `stdout-path`
        - serial core calls `of_console_check` to `add_preferred_console` for
          a port
        - if `earlycon` is specified on cmdline,
          `early_init_dt_scan_chosen_stdout` also parses the prop for earlycon
    - `of_aliases` points to `/aliases`
      - `of_alias_add` adds each alias to `aliases_lookup`
        - e.g., `serial2 = &uart3` is parsed as
          - `stem` is `serial`
          - `id` is 2
          - `np` is `uart3`
      - drivers use `of_alias_get_id` to look up id
        - e.g., this is useful to assign `ttyS2` to `uart3` deterministically
- `of_platform_default_populate_init` adds `platform_device`s for dt nodes
  - it creates platform devices for
    - certain `/reserved-memory` nodes
    - `/firmware` node
    - child of `/chosen` that is compatible with `simple-framebuffer`
    - direct children of the root node
    - grandchildren of the root node that are on certain buses
  - `of_platform_device_create` creates the platform device
    - `of_address_to_resource` parses `reg`

## Standard Properties

- `of_device_is_available` checks if `status` is okay
- `of_device_is_compatible` checks if a device is compatible with a string
  - it gets the `compatible` prop and performs strcmp
- `of_address_to_resource` parses `reg` and more to io resources
  - `of_get_address` returns `reg` value
    - most nodes are assumed to be on `default` bus
    - their parent nodes should have `#address-cells` and `#size-cells`
    - `size` is parsed from the size cells of `reg`
    - `flags` is assumed to be `IORESOURCE_MEM`
  - `of_translate_address` translates node addr to physical addr
    - it recursively translates the node addr to an addr relative to the
      parent node until it reaches the root node
    - `ranges` prop specifies the translation
- `of_dma_is_coherent` parses `dma-coherent`

## Interrupts

- `of_irq_get_byname` parses `interrupt-names` and `interrupts`
  - drivers typically call `platform_get_irq_byname` instead
    - `fwnode_irq_get_byname` calls `of_fwnode_property_read_string_array` to
      parse `interrupt-names` and calls `of_fwnode_irq_get` to parse
      `interrupts`

## Device Bindings

- schemas
  - <https://github.com/devicetree-org/dt-schema>
  - <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings>
- clk
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/clock/clock.yaml>
    - `#clock-cells` is typically 0, and specifier is omitted from `clocks`
    - `clock-names` is clock names
    - `clocks` is `(phandle, specifier)` pairs
  - `of_clk_get_hw` parses `clock-names` and `clocks`
    - drivers typically call `devm_clk_get` which calls `clk_get`
    - the clk driver should have called `of_clk_add_provider` to add a provider
  - `of_clk_set_defaults` parsed `assigned-clocks` and `assigned-clock-rates`
    - it is called from `platform_probe` as a clock consumer
      - `clk_set_rate` is called on the assigned clocks
- graph
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/graph.yaml>
    - the parent-child relations are not enough
    - phandles
    - ports and endpoints
- mbox
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/mbox/mbox-consumer.yaml>
    - `mbox-names` is mbox names
    - `mboxes` is `(phandle, specifier)` pairs
    - `shmem` is `phandle` array
  - `mbox_request_channel_byname` parses `mbox-names` and `mboxes` to return
    the mbox channel
- pci
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/pci/pci-bus-common.yaml>
    - `device_type` is `pci`
- phy
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/phy/phy-consumer.yaml>
    - `phy-names` is phy names
    - `phys` is `(phandle, specifier)` pairs
  - `phy_get` parses `phy-names` and `phys` to return the phy
- pinctrl
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/pinctrl/pinctrl-consumer.yaml>
    - `pinctrl-names` is state names
    - `pinctrl-%d` is `phandle`
  - `pinctrl_get` parses `pinctrl-names` and `pinctrl-%d` to convert each of
    them to a `pinctrl_map`
    - this is different from clk, mbox, etc.
    - e.g., a mmc controller might require a different state depending on the
      speed of the inserted sd card
- pmdomain
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/power-domain/power-domain-consumer.yaml>
  - `dev_pm_domain_attach_by_name` parses `power-domain-names` and
    `power-domains`
    - `genpd_get_from_provider` returns the power domain
    - the power controller driver should have called
      `of_genpd_add_provider_onecell` to register the power domain
- reset
  - <https://github.com/devicetree-org/dt-schema/blob/main/dtschema/schemas/reset/reset.yaml>
    - `reset-names` is reset names
    - `resets` is `(phandle, specifier)` pairs
  - `__of_reset_control_get` parses `reset-names` and `resets` to return the
    reset controller
    - it falls back to `reset-gpios`
- opp
  - <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings/opp/opp-v2-base.yaml>
  - `devm_pm_opp_of_add_table` parses `operating-points-v2` and opp table
    - `_of_add_opp_table_v2` parses the table
    - `_opp_add_static_v2` parses an entry
  - `dev_pm_opp_calc_power` parses `dynamic-power-coefficient`
    - the prop is the device capacitance and is used to estimate the power
      consumption at a given frequency and voltage (from opp)
- regulator
  <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings/regulator/regulator.yaml>
  - `of_get_regulator` parses `%s-supply`
    - drivers typically call `devm_regulator_get` which calls `regulator_get`
    - the regulator driver should have called `devm_regulator_register` to
      register the regulator
- node dependencies
  - all parent nodes
  - `clocks` and `clock-names` indiciate a clock consumer
    - the device requires some or all of the clocks to function
  - `mboxes` and `mbox-names` indicate a mailbox consumer
    - the device uses some or all of the mailboxes for communication
    - this is commonly used by coprocessors
  - `phys` and `phy-names` indicate a phy consumer
    - this happens with controllers that have connectors, such as usb
    - the device (controller) needs the phy (connector) to be useful
  - `pinctrl-%d` and `pinctrl-names` indicate a pinctrl consumer
    - an soc typically has more ip blocks than it has physical pins
    - when a device is an ip block, it might require the pinctrl to be
      configured in a certain way to function
  - `power-domains` and `power-domains-names` indicate a pmdomain consumer
    - the power rail to the device might need to be switched on for the device
      to function
  - `resets` and `reset-names` indicate a reset consumer
    - the device has reset pins connected to the reset controller
  - `%s-supply` indicates a regulator consumer
    - the device requires some or all of the regulators to function

## Example

- `Documentation/devicetree/bindings/gpu/arm,mali-valhall-csf.yaml`
  - node name must be `gpu@addr`
  - `compatible` must be `rockchip,rk3588-mali` or `arm,mali-valhall-csf`
    - a `const` is just an `enum` with a single possible value
  - `reg` has at most 1 item
  - `interrupts` has exactly 3 items
  - `interrupt-names` must be `job`, `mmu`, and `gpu`
  - `clocks` has 1 to 3 items
  - `clock-names` has 1 to 3 items, and must be `core`, `coregroup`, and `stacks`
  - `mali-supply` is a valid property
    - is this what `true` means?
  - `operating-points-v2` defaults to true
    - it is a phandle to `opp-table`
  - `opp-table` is an object
    - this is when `opp-table` is embedded
  - `power-domains` has 1 to 5 items
  - `power-domain-names` has 1 to 5 items
  - `sram-supply` is a valid property
  - `#cooling-cells` must be 2
  - `dynamic-power-coefficient` is a u32
  - `dma-coherent` is a valid property
  - required props
    - `compatible`, `reg`, `interrupts`, `interrupt-names`, `clocks`,
      `mali-supply`
- `rockchip/rk3588-base.dtsi` defines `gpu: gpu@fb000000`
  - `compatible = "rockchip,rk3588-mali", "arm,mali-valhall-csf";`
  - `reg = <0x0 0xfb000000 0x0 0x200000>;`
    - the root node sets `#address-cells` and `#size-cells` to `<2>`
    - `offset 0xfb000000 size 0x200000`
  - `#cooling-cells = <2>;`
  - `assigned-clocks = <&scmi_clk SCMI_CLK_GPU>;`
  - `assigned-clock-rates = <200000000>;`
  - `clocks = <&cru CLK_GPU>, <&cru CLK_GPU_COREGROUP>, <&cru CLK_GPU_STACKS>;`
  - `clock-names = "core", "coregroup", "stacks";`
  - `dynamic-power-coefficient = <2982>;`
  - `interrupts = <GIC_SPI 92 IRQ_TYPE_LEVEL_HIGH 0>, <GIC_SPI 93 IRQ_TYPE_LEVEL_HIGH 0>, <GIC_SPI 94 IRQ_TYPE_LEVEL_HIGH 0>;`
  - `interrupt-names = "job", "mmu", "gpu";`
  - `power-domains = <&power RK3588_PD_GPU>;`
  - `status = "disabled";`
- `rockchip/rk3588-opp.dtsi` modifies `&gpu`
  - `operating-points-v2 = <&gpu_opp_table>;`
- `rockchip/rk3588-rock-5b.dts` modifies `&gpu`
  - `mali-supply = <&vdd_gpu_s0>;`
  - `status = "okay";`
