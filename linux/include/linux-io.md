Kernel IO
=========

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

