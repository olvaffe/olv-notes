Kernel Driver Core
==================

## Early Drivers

- early drivers do NOT use the driver model
- why early drivers?
  - scheduler must be working for pid 0 `start_kernel` to switch to pid 1
    `kernel_init`
    - this needs hw timer, which needs clk and irq
  - console for debugging
    - this needs serial, which also needs clk and irq
    - there is an even earlier earlycon, which has no dependency
- early earlycon drivers
  - `setup_arch` calls `parse_early_param` to parse `earlycon` param, where
    drivers are defined by `OF_EARLYCON_DECLARE`
- early irqchip drivers
  - x86 provides them from the arch code
  - arm calls `irqchip_init`, where drivers are defined by `IRQCHIP_DECLARE`
- early clk drivers
  - arm calls `of_clk_init`, where drivers are defined by `CLK_OF_DECLARE`
  - `CLK_OF_DECLARE_DRIVER` defines a driver that clears `OF_POPULATED`
    - this allows the driver model to create a platform device for the node
      later and allows a full-featured platform driver to take over
- early timer drivers
  - x86 provides them from the arch code
  - arm calls `timer_probe`, where drivers are defined by `TIMER_OF_DECLARE`
- early console drivers
  - `start_kernel` calls `console_init`, where drivers are defined by
    `console_initcall`
  - full-featured driver-model drivers can take over later

## Initialization

- `devices_init` is called from `driver_init` during driver subsystem init
  - `devices_kset` is created with `kset_create_and_add` with no parent
    kobject (`/sys/devices`)
  - `dev_kobj` is created with `kobject_create_and_add` with no parent kobject
    (`/sys/dev`)
  - `sysfs_dev_block_kobj` is created with `kobject_create_and_add` under
    `dev_kobj` (`/sys/dev/block`)
  - `sysfs_dev_char_kobj` is created with `kobject_create_and_add` under
    `dev_kobj` (`/sys/dev/char`)
- if a device has a major, `device_add` calls
  - `device_create_sys_dev_entry` to create a symlink under
    `/sys/dev/{block,char}`
  - `devtmpfs_create_node` to create a node in devtmpfs
- `hypervisor_init` creates `hypervisor_kobj`, `/sys/hypervisor`

## Device Registration

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
  - `/sys/devices/breakpoint` from `init_hw_breakpoint`
  - `/sys/devices/cpu` from `init_hw_perf_events`
  - `/sys/devices/cstate_core` and `/sys/devices/cstate_pkg` from
    `cstate_init`
  - `/sys/devices/i915` from `i915_pmu_register`
  - `/sys/devices/intel_bts` from `bts_init`
  - `/sys/devices/intel_pt` from `pt_init`
  - `/sys/devices/msr` from `msr_init`
  - `/sys/devices/power` from `rapl_pmu_init`
  - `/sys/devices/software` from `perf_event_init`
  - `/sys/devices/tracepoint`, `/sys/devices/kprobe`, and
    `/sys/devices/uprobe` from `perf_tp_register`
  - `/sys/devices/uncore_*` from `uncore_pmu_register`
- `/sys/devices/isa` is by `isa_bus_init`
- `/sys/devices/LNXSYSTM:00` is by `acpi_device_add` on `ACPI_SYSTEM_HID`
- `/sys/devices/pci0000:00` is by `pci_register_host_bridge`
- `/sys/devices/platform` is by `platform_bus_init`
- `/sys/devices/pnp0` is by `pnp_register_protocol`
- `/sys/devices/wakeup0` is by `pm_autosleep_init`
- `/sys/devices/system` is a kset by `buses_init`
  - `cpu` is from `cpu_dev_init`
  - `memory` is from `memory_dev_init`
  - don't use `subsys_system_register` in new code
- `/sys/devices/virtual` is by `virtual_device_parent`
  - `workqueue` is from `wq_sysfs_init`, a virtual subsys
  - `dma_heap` is from `dma_heap_add`, a class device with no parent

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

## Device Link

- <https://docs.kernel.org/driver-api/device_link.html>
  - device hierarchy forms a tree
    - each child depends on its parent
  - device link turns device hierarchy into a DAG
    - each consumer depends on its producer(s)
- `device_link_add` adds a devlink
  - a `device_link` is also a `device`
    - `link->supplier` points to the supplier
    - `link->consumer` points to the consumer
    - link name is `<supplier-bus:name>--<consumer-bus:name>`
    - `link->status` depends on `supplier->links.status` and
      `consumer->links.status`
  - the new link is added to `supplier->links.consumers` and
    `consumer->links.suppliers`
  - `device_reorder_to_tail` moves the consumer to the tail to ensure
    suppliers always come before consumers
    - `devices_kset_move_last` updates `devices_kset`
    - `device_pm_move_last` updates `dpm_list`
- before a driver probes a dev, `really_probe` calls
  `device_links_check_suppliers` first to potentially defer the probe
  - if a supplier on `dev->links.suppliers` is not ready, it prints `supplier <foo> not ready`
- `dev->links.status`
  - `DL_DEV_NO_DRIVER` is the initial status
  - when `really_probe` calls `device_links_check_suppliers`, the status
    becomes `DL_DEV_PROBING`
  - after the driver probes, `driver_bound` calls `device_links_driver_bound`
    to set the status to `DL_DEV_DRIVER_BOUND`
  - if the driver fails to probe, `device_links_no_driver` resets the status
    to `DL_DEV_NO_DRIVER`
  - when `device_release_driver_internal` unbinds a driver, the status becomes
    `DL_DEV_UNBINDING` after `device_links_busy` returns, and then becomes
    `DL_DEV_NO_DRIVER` after `device_links_driver_cleanup` returns
- `link->status`
  - `DL_STATE_NONE` means not tracked
  - `DL_STATE_DORMANT` means supplier is not ready (no driver bound)
  - `DL_STATE_AVAILABLE` means supplier is ready but consumer is not
  - `DL_STATE_CONSUMER_PROBE` means supplier is ready and consumer is probing
  - `DL_STATE_ACTIVE` means both supplier/consumer are ready
  - `DL_STATE_SUPPLIER_UNBIND` means supplier is unbinding
  - when `device_link_add` adds a link,
    - if `DL_FLAG_STATELESS` (no tracking), the status is `DL_STATE_NONE`
    - otherwise, `device_link_init_status` sets the status based on
      consumer/supplier status
- `link->flags`
  - `DL_FLAG_STATELESS` means explicit link lifetime management
  - `DL_FLAG_AUTOREMOVE_CONSUMER` means the link is auto-removed when the consumer
    driver unbinds
  - `DL_FLAG_PM_RUNTIME` means the link participates in runtime pm
  - `DL_FLAG_RPM_ACTIVE` means the link keeps the supplier powered
  - `DL_FLAG_AUTOREMOVE_SUPPLIER` means the link is auto-removed when the
    supplier driver unbinds
  - `DL_FLAG_AUTOPROBE_CONSUMER`
  - `DL_FLAG_MANAGED` is the opposite of `DL_FLAG_STATELESS`
  - `DL_FLAG_SYNC_STATE_ONLY`
  - `DL_FLAG_INFERRED` means the link is auto-derived from dt
  - `DL_FLAG_CYCLE` means cyclic dep between consumer/supplier
- `device_links_driver_bound` is called on a dev after a driver probes the dev
  - for each devlink from the dev to its consumer,
    - `link->status` transitions from `DL_STATE_DORMANT` to
      `DL_STATE_AVAILABLE`
    - if `DL_FLAG_AUTOPROBE_CONSUMER`, `driver_deferred_probe_add` the
      consumer
  - for each devlink from a supplier to this dev,
    - `link->status` transitions from `DL_STATE_CONSUMER_PROBE` to
      `DL_STATE_ACTIVE`
  - `dev->links.status` transitions to `DL_DEV_DRIVER_BOUND`

## `fw_devlink`

- `fw_devlink` creates devlinks automatically based on fwnodes
- `device_add` calls `fw_devlink_link_device` on each device
  - `fw_devlink_parse_fwtree` calls `add_links` cb, which points to
    `of_fwnode_add_links`, on the fwnode
    - it checks dt node properties matched by `of_supplier_bindings`, which
      consists of `clocks`, `interconnects`, `mboxes`, `power-domains`,
      `nvmem-cells`, `phys`, `pinctrl-*`, `pwms`, `resets`, `leds`,
      `backlight`, `panel`, `*-supply`, `*-gpio`, etc.
    - this fwnode is the consumer
    - `fwnode_link_add` adds an `fwnode_link`
      - `link->supplier` points to the supplier
      - `link->consumer` points to the consumer
      - the link is added to `supplier->consumers` and `consumer->suppliers`
  - `__fw_devlink_link_to_consumers` creates devlinks to consumers automatically
    - for each consumer fwnode, if the consumer device has been added,
      `fw_devlink_create_devlink` calls `device_link_add` for the
      supplier-consumer device pair
    - `__fwnode_link_del` removes fwnode links if they have been promoted to
      devlinks
  - `__fw_devlink_link_to_suppliers` creates devlinks to suppliers automatically
    - for each supplier fwnode, if the supplier device has been added,
      `fw_devlink_create_devlink` calls `device_link_add` for the
      supplier-consumer device pair
    - `__fwnode_link_del` removes fwnode links regardless whether they have
      been promoted to devlinks
- fwnode flags
  - `FWNODE_FLAG_LINKS_ADDED`
    - when `device_add` calls `fw_devlink_link_device`, the flag ensures
      `add_links` is called only once for each fwnode to add fwnode
      supplier/consumer links
  - `FWNODE_FLAG_NOT_DEVICE`
    - when `fw_devlink_create_devlink` adds a devlink, the flag causes the
      parent device of the supplier to be used as the supplier
  - `FWNODE_FLAG_INITIALIZED`
    - early drivers do not bind to their devices but instead set this flag
  - `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD`
    - net mdio sets the flag to prevent `fw_devlink_create_devlink` from adding
      devlink
  - `FWNODE_FLAG_BEST_EFFORT`
    - earlycon `stdout-path` sets the flag to prevent
      `device_links_check_suppliers` from checking the supplier deps
  - `FWNODE_FLAG_VISITED`
    - when `fw_devlink_create_devlink` calls `__fw_devlink_relax_cycles` to find
      dep cycles, the flag ensures each fwnode is visited once
- `device_links_check_suppliers` also checks for fw devlinks
  - if `fwnode_links_check_suppliers` returns a fwnode, it prints `wait for supplier <foo>`
    - this happens when `device_add` hasn't been called for the supplier dt
      node and thus the fwnode link hasn't been removed
- 10 seconds after the last driver loaded,
  - `deferred_probe_timeout_work_func` is called after
    `driver_deferred_probe_timeout` (10s)
  - `fw_devlink_drivers_done` relaxes devlinks where the supplier driver is
    missing
    - `dev->can_match` is true if any driver matches (not probes) the dev
    - `FW_DEVLINK_FLAGS_PERMISSIVE` essentially means
      `DL_FLAG_SYNC_STATE_ONLY` and allows `device_links_check_suppliers` to
      ignore the link
  - `driver_deferred_probe_trigger` probes all deferred devices one last time
  - it prints a warning for remaining deferred devices
  - `fw_devlink_probing_done` prints a warning for each devlink where the
    consumer fails to probe
    - `device_links_driver_bound` has called `dev_sync_state` on each dev
      whose consumers have all been probed
