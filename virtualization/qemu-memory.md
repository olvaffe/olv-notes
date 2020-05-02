# QEMU Memory API

## `MemoryRegion`

* A `MemoryRegion` models the memory, I/O buses, and controllers of a virtual
  machine.
* There are several types of `MemoryRegion`
  * RAM, `memory_region_init_ram`
    * backed by host memory
  * MMIO, `memory_region_init_io`
    * backed by host callbacks
  * ROM, `memory_region_init_rom`
    * similar to RAM, but read-only
  * ROM device, `memory_region_init_rom_device`
    * similar to ROM, but writes are backed by host callbacks just MMIO
  * IOMMU region, `memory_region_init_iommu`
    * forward accesses to another `MemoryRegion`
  * container, `memory_region_init`
    * a container for other `MemoryRegion`s
    * e.g., a PCI BAR may be composed of a RAM and an MMIO `MemoryRegion`s
  * alias, `memory_region_init_alias`
    * a view into another `MemoryRegion`
    * e.g., 32-bit guest with >4GB RAM
  * reservation region, `memory_region_init_io`
    * MMIO with no callbacks
    * e.g., trace address space that is handled by KVM
  * It is possible to add subregions to a region, making the region a
    container.  But it is not recommended.

## `AddressSpace`

* An `AddressSpace` describes a mapping of addresses to `MemoryRegion`s
* Each emulated CPU has an `AddressSpace`.  Some CPUs have more than one
  `AddressSpace` (e.g., one for Secure and one for NonSecure).
* Emulated devices that are capable of DMAs also have `AddressSpace`s.
* For legacy reason, there is also a system `AddressSpace` that has all
  devices and memories.

## `MemoryListener`

* Listen to changes to physical address space
  * `region_add`
  * `region_del`
