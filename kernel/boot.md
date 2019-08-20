Kernel Boot on x86-64
=====================

## `boot/compressed/`

* `vmlinux` in the root is objcopy'ed to `boot/compressed/` as `vmlinux.bin`,
  and is stripped
* `vmlinux.bin` is gzipped as `vmlinux.bin.gz`.
* create `piggy.o` with `vmlinux.bin.gz` embedded with the help of `mkpiggy`
* `piggy.o` and other sources under the directory are compiled into `vmlinux`,
  using `vmlinux.lds`.

## `boot/`

* `vmlinux` under `compressed/` is objcopy'ed here as `vmlinux.bin`.
* `setup.elf` is compiled from sources under this directory, using `setup.ld`.
* `setup.bin` and `vmlinux.bin` are put together by `tools/build` to create
  `bzImage`.
  * `build` patches the header to give correct values.

## Bootloader

* `Document/x86/boot.rst`
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
* After entering protect-mode, it jumps to the start of protect-mode code at
  `code32_start`

## Protect-Mode and `head_64.S`

* It usually starts from `startup_32` in `boot/compressed/head_64.S` to
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

## 64-bit mode and another `head_64.S`

* Ok, it starts from `startup_64` in `kernel/head_64.S`.
  * with identity-mapped address space
* And it jumps to `x86_64_start_kernel`, which calls `start_kernel`.
  * It is in 64-bit mode and paging is enabled.

## Stack

* The stack is set up by `movq initial_stack(%rip), %esp` in `kernel/head_64.S`.

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

## Resources

* `/proc/iomem`
* During `setup_arch`, `e820__reserve_resources` is called to turn
  non-reserved e820 entries into resources.
  * Incidentally, `e820__setup_pci_gap` decides `pci_mem_start`, which finds a gap
    between e820 entries that is high and large enough.
* In subsys initcalls, `pcibios_init` is called.  It calls
  `pcibios_resource_survey` to allocate PCI resources.  That calls
  `e820__reserve_resources_late` to insert the rest of e820 entries.

## Kernel

* Linux x86 boot protocol is defined in `Documentation/x86/boot.rst`
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
  * In `kernel_init_freeable`, if `/init` exists (from initramfs), it runs the
    command.
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
