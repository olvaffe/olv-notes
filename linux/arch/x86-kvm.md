X86 KVM
=======

## Nested Paging

- AMD-V Nested Paging White Paper
  - <http://developer.amd.com/wordpress/media/2012/10/NPT-WP-1%201-final-TM.pdf>
- SW shadow page table
  - guest has a guest page table
  - hypervisor has a shadow page table
  - the HW MMU uses the shadow page table when the guest is active
  - the guest page table is marked read-only
    - whenever the guest updates it, it traps into the hypersor
    - the hypervisor updates both the guest and the shadow page tables
- HW nested page table
  - HW MMU uses the guest page table directly
  - an additional nested page table set up by the hypervisor is used to
    translate guest physical address to host physical address
  - for a 4-level paging walk in guest results in 5 walks in the nested page
    walker (to access PML4, PDPE, PDE, PTE, and the page)
- memory type selection
  - MTRR
    - 0x0: UC
    - 0x1: WC
    - 0x4: WT
    - 0x5: WP (write protected)
    - 0x6: WB
    - Linux does not use MTRR.  BIOS(?) sets MTRRdefType to 0xc06, which means
      WB by default.
  - `IA32_PAT` MSR has 8 3-bit page attribute fields, PA0..PA7
    - each field can have one of these values
      - 0x0: UC
      - 0x1: WC
      - 0x4: WT
      - 0x5: WP
      - 0x6: WB
      - 0x7: UC-
    - In `pat_init`, they are initialized to
      - PA0: WB
      - PA1: WC
      - PA2: UC-
      - PA3: UC
      - PA4: WB (unused)
      - PA5: WP
      - PA6: UC- (unused)
      - PA7: WT
  - PTE's bit 7 (PAT), 4 (PCD), and 3 (PWT) form a 3-bit index that is used to
    select from PA0..PA7
    - `pgprot_writecombine` maps to PA1
  - In EPT (extended page table, used by KVM), MTRR is ignored and EPT bits
    5:3 replace MTRR (0: UC, 1: WC, 4: WT, 5 WP, 6: WB)
    - `vmx_get_mt_mask`

