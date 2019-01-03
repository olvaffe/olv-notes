Kernel Boot on x86-32
=====================

## `boot/compressed/`

* `vmlinux` in the root is copied to `boot/compressed/` as `vmlinux.bin`.
* `vmlinux.bin` is gzipped as `vmlinux.bin.gz`.
* `vmlinux.bin.gz` is compiled into `piggy.o` using `vmlinux.scr`.
* `piggy.o` and other sources under the directory are compiled into `vmlinux`,
  using `vmlinux.lds`.

## `boot/`

* `vmlinux` under `compressed/` is copied here as `vmlinux.bin`.
* `setup.elf` is compiled from sources under this directory, using `setup.ld`.
* `setup.bin` and `vmlinux.bin` are put together by `build` to create `bzImage`.
  * `build` patches the header to give correct values.

## Bootloader

* `Document/x86/boot.txt`
* Bootloader loads first some KB of the kernel to a place it chooses.
  * It contains the header and real-mode code.
* Bootloader also loads protect-mode code to the right place.
  * The right place is given by `code32_start` of the header
  * It is usually in the high memory, which is not accessible in real mode
  * So the bootloader might either enter protected mode
  * Or, use `Int 15/AH=87h`, <http://www.ctyme.com/intr/rb-1527.htm>
* It then jumps to the start of the header.

## Real-Mode

* It starts from `boot/header.S`, which jumps to `main` in `boot/main.c`.
* After entering protect-mode, it jumps to the start of protect-mode code.

## Protect-Mode and `head.[cS]`

* It usually starts from `startup_32` in `boot/compressed/head_32.S` to
  decompress the kernel.
  * The compressed kernel is loaded at `code32_start` by current bootloader,
    which is hardcoded at `0x100000`.  Future bootloader should load it to
    `pref_address`, which is `LOAD_PHYSICAL_ADDR`.
  * The decompressor wants to decompress it to `LOAD_PHYSICAL_ADDR` for
    non-relocatable kernel or decompress in-place for relocatable kernel.
  * The decompressed kernel is an elf image.  It is parsed and its elf programs
    are copied in-place to the right place.
  * It has in `ebp` the loaded address (`code32_start` or `pref_address`) and in
    `ebx` the `LOAD_PHYSICAL_ADDR` (usually the same, 0x1000000 since 2.6.31).
    The size difference between the uncompressed/compressed images plus
    necessary decompress offset is in `z_extract_offset`.
  * It copies the compressed kernel from loaded address to physical address
    (`ebx`).  It is copied backward to allow in-place copy.  It is copied to the
    end of physical address to allow in-place decompressions.
* Ok, it starts from `startup_32` in `kernel/head_32.S`.
  * In `default_entry`, `swapper_pg_dir` stores PDEs and `__brk_base` stores
    PTEs.  It is identify map, except for the last entry of PDE which points to
    `swapper_pg_fixmap` for fixmap.
    * The kernel address space is 1G, while each PDE addresses 4M.  It is going
      to need `1G / 4M = 256` PTE tables to map the low memory in
      `init_memory_mapping`.
    * `MAPPING_BEYOND_END` gives the size needed to store these PTE tables.
    * Here, we only need to map enough memory for kernel image and the PTE
      tables.
    * In each loop, we create a PTE table (4M).  Two PDE entries will point to
      this table, one for identity map, one for kernel linear map
  * `max_pfn_mapped` stores the max pfn that is mapped.
  * Usually, the first 8MB is mapped.  It is so that we can access kernel
    variables after enabling paging.
* And it jumps to `i386_start_kernel`, which calls `start_kernel`.
  * It is in protect-mode and paging is enabled.
  * `i386_start_kernel` calls `reserve_early` to mark the region of kernel image
    as reserved in e820.
  * It also reserves the ramdisk, although it might not be accessible (only the
    first 8MB of the memor is mapped).

## brk section

* Introduced in `93dbda7cbcd70a0bd1a99f39f44a9ccde8ab9040`, 2009.
  * And modified several times after.
* In the linker script, `.brk` is reserved for `*(.brk_reservation)` and
  is delimited by `__brk_base` and `__brk_limit`.  It comes just after `.bss`
  and just before `.end`.
* `RESERVE_BRK` is used to reserve brk space
  * It creates `.brk_reservation` section with the given size
  * It is used to reserve space for init page tables and dmi
* When `head_32.S` creates `default_entry`, it creats page tables in `.brk`.
  * `_brk_end` marks the location after the page tables.
  * `extend_brk` is used to alloc space in brk.  It extends `_brk_end`.
    * It is used indirectly by `dmi_alloc` in `drivers/firmware/dmi_scan.c`.
* `reserve_brk` is called to reserve brk as `reserve_early`.  It also pins the
  brk as read-only.
  * It reserves up to `_brk_end`, not to `_brk_limit`.  Therefore, it is safe to
    `RESERVE_BRK` a large (safe) region.

## Memory Initialization

* x86-32 uses `include/asm/pgtable-2level.h`, 2-level page table.
  * `PGDIR_SHIFT == PUD_SHIFT == PMD_SHIFT == 22`.
  * `PTRS_PER_PGD` is 1024, `1 << (32 - 22)`.
  * `PTRS_PER_PTE` is 1024, `1 << (22 - 12)`.
* There are three memory allocators
  * brk, for initial page tables and dmi
  * e820 and `alloc_low_page`.  It is only used in `mm/init_32.c`.
  * bootmem,
  * buddy allocator
* In `start_kernel`, after the linux banner is printed, `setup_arch` is called.
  * It calls `setup_memory_map` to decide physical memory maps.
  * It calls `e820_end_of_ram_pfn` to decide 
    * `max_pfn`, (pfn of the last page of the DRAM) + 1.
    * `max_arch_pfn`, `((1 << 32) >> 12)`
  * It calls `find_low_pfn_range` to decide
    * `max_low_pfn`, which is `min(max_pfn, MAXMEM_PFN)`.  `MAXMEM_PFN` is the
      1G kernel address space minus the reserved regions (usually 887MB).
    * `highmem_pages`, which is `max_pfn - MAXMEM_PFN`.
  * It calls `reserve_brk` to resert brk region.
  * It calls `init_memory_mapping` to map the low memory.
    * It runs before bootmem for early direct access.
    * The first range is the first 4MB (`PMD_SIZE`), using 4K page.
    * The second range is the rest, using 4M page
    * It calls `find_early_table_space` to reserve in `e820_table_start` and
      `e820_table_end` a place to store the page tables.
      * This storage is available to others through `alloc_low_page`.  It is
        the `MAPPING_BEYOND_END` mapped in `head_32.S` 
      * This storage is reserved by `reserve_early`.
    * The mappings are set up by `kernel_physical_mapping_init`.
    * cr3 is loaded, with `swapper_pg_dir` still being the PDE table.
    * `max_low_pfn_mapped` and `max_pfn_mapped` are set to `max_low_pfn`
      if everything goes normally.
    * That is, the mapped memory goes from 8MB to whole low memory.
  * It calls `initmem_init` to set up bootmem, boot-time physical memory
    allocator.
    * `e820_register_active_regions` is called with node 0 and all physical
      memory.  All e820 ram regions are stored in `early_node_map` of buddy
      allocator.
    * `setup_bootmem_allocator` is called.  It sets `after_bootmem` to 1.
    * It `reserve_early` an area for `bootmap`.
    * Later, `early_res_to_bootmem` is called to migrate early reserved area to
      bootmem map.
  * It calls `paging_init` to allocate an array of all `struct page *` from
    bootmem.
    * All pages are initialized as reserved at this point by `memmap_init_zone`.
* Much later in `start_kernel`, `mem_init` is called
  * It calls `free_all_bootmem` to return unused pages in bootmem to the buddy
    allocator.
  * `zap_low_mappings` is called to, forget low memory mappings.  That is,
    virtual address `< PAGE_OFFSET` is no longer mapped.  This is so that one
    can only access kernel memory from kernel address space.
* An example of early reserved regions

    (8 early reservations) ==> bootmem [0000000000 - 00377fe000]
      #0 [0000000000 - 0000001000]   BIOS data page ==> [0000000000 - 0000001000]
      #1 [0000001000 - 0000002000]    EX TRAMPOLINE ==> [0000001000 - 0000002000]
      #2 [0000006000 - 0000007000]       TRAMPOLINE ==> [0000006000 - 0000007000]
      #3 [0000100000 - 00004cb180]    TEXT DATA BSS ==> [0000100000 - 00004cb180]
      #4 [00004cc000 - 00004cf000]    INIT_PG_TABLE ==> [00004cc000 - 00004cf000]
      #5 [000009fc00 - 0000100000]    BIOS reserved ==> [000009fc00 - 0000100000]
      #6 [0000007000 - 0000008000]          PGTABLE ==> [0000007000 - 0000008000]
      #7 [0000008000 - 000000f000]          BOOTMAP ==> [0000008000 - 000000f000]
  * PGTABLE is one page because only the first 4M uses 4K page.  The rest uses
    4M page and does not need to allocate memory.

## Resources

* `/proc/iomem`
* During `setup_arch`, `e820_reserve_resources` is called to turn non-reserved e820
  entries or entries located in the first 1MB into resources.
  * Incidentally, `e820_setup_gap` decides `pci_mem_start`, which finds a gap
    between e820 entries that is high and large enough.
* In subsys initcalls, `pcibios_init` is called.  It calls
  `pcibios_resource_survey` indirectly to allocate PCI resources.  It also calls
  `e820_reserve_resources_late` to insert the rest of e820 entries.

## Stack

* The stack is set up by `lss stack_start,%esp` in `head_32.S`.
  * `ss` is set to `__BOOT_DS`
  * `%esp` is set to `init_thread_union+THREAD_SIZE`.  It is so that
    `current_thread_info` works.
  * Accidentally, the `init_thread_union` (or, more accurately `init_task`)
    becomes the idle thread in `rest_init`.

## Kernel

* Linux x86 boot protocol is defined in `Documentation/x86/boot.txt`
  * the bootloader should load the kernel to certain memory addresses
  * the first sector of the kernel is used for communications between the
    bootloader and the kernel: how big is the kernel?  where did the bootloader
    write the commandline to?  where was the initramfs loaded?
* After arch specific code, `start_kernel` in `init/main.c` is called.  It
  spawns a thread to run `kernel_init`
  * In `do_initcalls`, `populate_rootfs` is called.  It unpacks the the internal
    initramfs to the rootfs, which is usually empty.  It then loads the external
    initramfs as loaded by the bootloader.  See
    `Documentation/filesystems/ramfs-rootfs-initramfs.txt`
  * In `init_post`, if `/init` exists (from initramfs), it runs the command.
  * If no initiramfs, it calls `prepare_namespace` to mount the root device and
    runs `/sbin/init`.  For `root=/dev/sda1`, it is translated to major/minor.
    `/dev/root` is created using the major/minor and is used for mounting.
    In `mount_block_root`, the device is mounted to `/root` and the kernel chdir
    to `/root`.  Finally, the mount point is moved to `/`.
  * Otherwise, see next section

## initramfs

* An initiramfs can be unpacked using
  `$ gunzip -c /boot/initrd.img-3.2.0-2-amd64 | cpio -idv`
* `/init` is executed.  It parses the kernel cmdline and does many other things.
  Among them,
  * it sources `/scripts/local` and run `mountroot` to mount the root
* finally, it calls `/sbin/init` of the root device
