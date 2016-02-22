Kernel Driver 123
=================

## Device Model

* `/sys/devices` is hierarchical.
* The others might not.  The goal of other directories is not to give a
  hierarchical view of the device tree.
  * E.g. `/sys/bus/pci/devices` contains all pci devices in a flat view.
* A SDHCI host controller lives on the PCI bus and has class mmc_host.  The card
  inserted lives on mmc bus, and is a child of the mmc_host?
* `struct device` has
  * `struct device *parent`, the device's parent
  * `struct device_private *p`, for deivce core use, like the device's children
  * `struct device_type *type`, the type of the device
  * `struct bus_type *bus`, the type of bus the device is on
  * `struce device_driver *driver`, the driver of the device
  * `struct class *class`, the class of the device
* A device with `class` is called a class device.  When a class device is
  added, it is put under a subdirectory with its class name of its parent,
  unless its parent is also a class device.
  * E.g. `pcspkr` has no class and `platform` is its parent.  It will be
    found under `/devices/platform/pcspkr`.
  * E.g. `hwmon0` is a hwmon class device and it has no parent.
    It will be found under `/devices/virtual/hwmon/hwmon0`.
  * E.g. `hwmon1` is a hwmon class device and `coretemp.0` is its parent.  It
    will be found under `/devices/platform/coretemp.0/hwmon/hwmon1`.
* Where a device is put is decided by `get_device_parent`.
  * It decides the device's kobject parent.
  * the device's `class` and `parent` can be null.
  * if both of them is null, the device has no kobject parent.
  * if `class` is null and `parent` is non-null, the parent's kobject is the
    device's kobject parent.
  * if `class` is non-null, the kobject parent is
    * if the device has a `parent` and the `parent` has a `class`, then the
      parent's kobject is the device's kobject parent.
    * if there is no `parent`, `virtual` is used as a parent without `class`.
    * if the `parent` has no `class`, a new kobject is created and added as the
      parent's child and as the device's kobject parent.
* It should be noted that what is in the sysfs are decided by `kset` and
  `kobject`.
  * The hierarchy is not decided by device and its parent.  Otherwise, a device
    without class or parent would be found under `/`, not `/devices`.
  * A `kobject` is a directory holding files.
  * A `kset` is a `kobject` that can also hold `kobject`s.
* There are `struct device platform_bus` and `struct bus_type platform_bus_type`.
  * `platform_bus` has no class, nor parent, nor bus.  It can be found under
    `/devices/platform`.
  * `platform_bus_type` is not a device.  It has a `subsys` kobject whose kset
    is `bus_kset`, and this makes it accessible under `/bus/platform`.  It has
    two ksets under `subsys`, which correspond to `/bus/platform/devices` and
    `/bus/platform/drivers`.
  * Bus is used for device-driver matchings.
* Take a look at SDHCI controller attached to the PCI bus.
  * There is a PCI device `/sys/devices/pci0000:00/0000:00:1e.0/0000:03:0b.3`.
  * When its driver, `sdhci_pci`, is attached, it creates a mmc_host class
    device whose parent is the pci device
    `/devices/pci0000:00/0000:00:1e.0/0000:03:0b.3/mmc_host/mmc0`
  * When an SD card is inserted, the card is a device with parent mmc0
    `/devices/pci0000:00/0000:00:1e.0/0000:03:0b.3/mmc_host/mmc0/mmc0:e624`.
    * It lives on the `mmc` bus `/bus/mmc/devices/mmc0:e624`.
* It seems that class devices are usually functions of a physical device.
  * Those not under this category can be found under `/devices/virtual`.

## Random Notes

* open fileoperation has prototype `int (*open) (struct inode *, struct file *)`
  `struct inode` is the file on the fs, while `struct file` is a fd
* char node major number is a limited resouce.  `register_chrdev_region` to
  reserve a range.

## Memory Access

* CPU/system/real memory could be expressed in:
  * physical address: physical
  * virtual address: the way cpu sees it
  * bus address: the way a bus sees it
* Physical and bus addresses coincide on x86.
* PCI/io/shared memory is usually not DRAM (hardware-wise).  It is _NOT_ in the
  same address space as system memory is.  It should be accessed through
  `readb/writeb`.
  * It is in the same address space as system memory is on x86. 
* ISA/LPC DMA
 * `Documentation/DMA-ISA-LPC`
 * depend on `ISA_DMA_API`
 * `GFP_DMA` is for ISA DMA.
 * do not use `isa_virt_to_phys` because it depends on ISA

## Locking

* `local_irq_save` to avoid race with current cpu
* `spin_lock` to avoid race with another cpu
* Documentation/spinlocks.txt
* Concurrency: Two or more functions could be called concurrently.
* Reentrancy: The same function could be concurrently called.
* Mutex (kernel/mutex.c)
  * sleep with `__set_task_state(task, TASK_UNINTERRUPTIBLE); schedule();`
  * needs to be waked up

## TLB

* cachetlb.txt is a good read

## Module

* device tables are extracted from .o by `scripts/mod/file2alias.c`.  They are
  parsed and aliases are added to the final `.ko`.
