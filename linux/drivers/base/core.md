Kernel Driver Core
==================

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

## Device Link

- <https://docs.kernel.org/driver-api/device_link.html>
- device hierarchy forms a tree
  - each child depends on its parent
- device link turns device hierarchy into a DAG
  - each consumer depends on its producer(s)

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

## devlink

- dt devlinks
  - when `of_platform_default_populate_init` calls `device_add` to populate
    the platform bus with dt devices, `fw_devlink_link_device` adds devlinks
  - `fw_devlink_parse_fwtree` calls `add_links` cb, which points to
    `of_fwnode_add_links`, to add fwnode links
    - it checks dt node properties matched by `of_supplier_bindings` and adds
      them as suppliers
    - these properties are matched: `clocks`, `interconnects`, `mboxes`,
      `power-domains`, `nvmem-cells`, `phys`, `pinctrl-*`, `pwms`, `resets`,
      `leds`, `backlight`, `panel`, `*-supply`, `*-gpio`, etc.
  - `__fw_devlink_link_to_consumers` and `__fw_devlink_link_to_suppliers`
    adds devlinks
    - as devlinks are added, `__fwnode_link_del` removes fwnode links
- when `device_link_add` adds a devlink,
  - `device_reorder_to_tail` moves the consumer to the tail to ensure
    suppliers always come before consumers
    - `devices_kset_move_last` updates `devices_kset`
    - `device_pm_move_last` updates `dpm_list`
- before a driver probes a dev, `really_probe` calls
  `device_links_check_suppliers` first to potentially defer the probe
  - if `fwnode_links_check_suppliers` returns a fwnode, it prints `wait for supplier <foo>`
    - this happens when `device_add` hasn't been called for the supplier dt
      node and thus the fwnode link hasn't been removed
  - if a supplier on `dev->links.suppliers` is not ready, it prints `supplier <foo> not ready`
- `late_initcall(deferred_probe_initcall)`
  - it probes all devices that are on `deferred_probe_pending_list`
  - it schedules `deferred_probe_timeout_work_func` after
    `driver_deferred_probe_timeout` (10) seconds
  - when `deferred_probe_timeout_work_func` runs,
    - `fw_devlink_drivers_done` relaxes supplier requirement if the supplier
      has no driver at all
    - it probes all devices again
    - it prints `deferred probe pending: <reason>` for all remaining pending
      devs
    - `fw_devlink_probing_done`
      - `fw_devlink_dev_sync_state` prints `sync_state() pending due to <consumer>`
        - we want to call `dev_sync_state` on the supplier after all consumers
          are probed
        - if any consumer is not probed, this msg is printed
