ARM64
=====

## Bootloader

- bootloader has four responsibilities
  - initialize RAM
  - load device tree blob (.dtb)
  - load decompressed kernel image
  - jump to kernel image
- the kernel image has a 64-byte header
  - the first 8 bytes are instructions to jump to stext
  - there is `text_offset` and the image must be loaded to any 2MB base plus
    `text_offset`
- before jumping to the kernel,
  - reg x0 must contain the address of the loaded dtb
  - MMU must be off
  - more
- bootloader can patch the dtb
  - for serial number
  - machine revision
  - reserved memory
  - etc

## Kernel Image Addresses

- assembly
  - `ldr` loads the an offset
  - `adr` loads the addr of a PC-relative offset
    - the offset must be small-ish
  - `adrp` loads the addr of a PC-relative page-aligned offset
    - the offset can be huge but should be page-aligned
- `vmlinux.lds.S`
  - `KERNEL_START` is the va of `_text` symbol
  - `KERNEL_END` is the va of `_end` symbol
- kernel image va
  - `CONFIG_RELOCATABLE` is set and the bootloader can load the image to any
    2MB-base
  - `KIMAGE_VADDR` is the default va of `_text` symbol
  - `kimage_vaddr` is the real va of `_text` symbol
  - `kimage_voffset` is the offset from pa to va

## Memory Initialization

- by default
  - `CONFIG_ARM64_VA_BITS_48=y`
  - `CONFIG_ARM64_4K_PAGES=y`
  - `CONFIG_PGTABLE_LEVELS=4`
- `vmlinux.lds.S` reserves spaces for early page tables
  - `init_idmap_pg_dir`
    - this reserves `INIT_IDMAP_DIR_SIZE`, which is big enough for all levels
      of page tables to cover the kernel image, fdt, and swapper block
  - `init_pg_dir`
    - this reserves `INIT_DIR_SIZE`, which is big enough for all levels of
      page tables to cover the kernel image
    - `INIT_MM_CONTEXT` statically sets `init_mm::pgd` to `init_pg_dir`
  - `reserved_pg_dir` is used when no access is expected
  - `swapper_pg_dir`
  - `idmap_pg_dir`
  - `tramp_pg_dir`
- the kernel image header starts with` b primary_entry` to jump to
  `primary_entry`
- `init_kernel_el`
  - loads `INIT_SCTLR_EL1_MMU_OFF` to `SCTLR_EL1` to disable MMU
- `create_idmap` (in `head.S`)
  - initializes `init_idmap_pg_dir` with identity read-only mapping for the pa
    that covers the kernel image, fdt, and swapper block
  - modifies `init_idmap_pg_dir` such that the mapping for `init_pg_dir`, fdt,
    and swapper block is read-write
- `__cpu_setup`
  - loads `TCR_EL1` such that
    - when va bit48..63 are 0, use `TTBR0_EL1` (for user addresses)
    - when va bit48..63 are 1, use `TTBR1_EL1` (for kernel addresses)
  - sets `x0` to `INIT_SCTLR_EL1_MMU_ON`
- `__primary_switch`
  - sets
    - `x1` to `reserved_pg_dir`
    - `x2` to `init_idmap_pg_dir`
  - `__enable_mmu`
    - loads `x2` `TTBR0_EL1` (`x2` is `init_idmap_pg_dir`)
    - loads `x1` `TTBR1_EL1` (`x1` is `reserved_pg_dir`)
    - loads `x0` `SCTLR_EL1` (`x0` is `INIT_SCTLR_EL1_MMU_ON`)
  - `clear_page_tables` zeros out `init_pg_dir`
  - `create_kernel_mapping` initializes `init_pg_dir` with lienar read-write
    mapping for the pa that covers the kernel image
  - loads `init_pg_dir` to `TTBR1_EL1`
    - for a 48-bit va `xyz`, `0x0000xyz` is translated by `init_idmap_pg_dir`
      and `0xfffffxyz` is translated `init_pg_dir`
    - because the mappings are identity/linear, va `xyz` is translated to pa
      `xyz`
    - why do we need idmap?  because we use physical addresses before mmu and
      must use idmap after mmu
- `__primary_switched`
  - `early_fdt_map` remaps the fdt to `early_fdt_ptr`
    - `early_fixmap_init`
      - fixmap is a small region (a few MBs) of va before `VMEMMAP_START`
      - it can be used to map pa outside of the kernel image, such as fdt
      - see `fixed_addresses`
  - `start_kernel`
- `setup_arch`
  - `cpu_uninstall_idmap` switches `TTBR0_EL1` from `init_idmap_pg_dir` to
    `reserved_pg_dir`
  - `paging_init` migrates from `init_pg_dir` to `swapper_pg_dir`
    - `swapper_pg_dir` is initialized by `map_kernel` and `map_mem`
    - `cpu_replace_ttbr1` switches `TTBR1_EL1` from `init_pg_dir` to
      `swapper_pg_dir`
    - `init_mm.pgd` switches from `init_pg_dir` to `swapper_pg_dir`
    - pages used by fixmap and `init_pg_dir` are freed
    - `idmap_pg_dir` is intialized by `create_idmap`
      - this is used rarely
      - when a cpu starts up (e.g., a non-boot cpu or on cpu resume), mmu is
        disabled and pa is used.  We need idmap when mmu is enabled, until we
        stop using pa
- it looks like, after boot initialization,
  - kernel addresses are mapped by `swapper_pg_dir`
  - user addresses are mapped by user page table
    - but if there is no user context, user addresses are mapped by
      `reserved_pg_dir`

## Kernel Memory Layout

- user va (bits 48..63 are 0) is mapped by `TTBR0_EL1`
  - when there is a user context, this is the user mm
  - when there is no user context, this is `reserved_pg_dir`
  - idmap
    - on cpu startup or resume, mmu is disabled and we work with pa, which is
      in the user va region
    - to be able to enable mmu, we must set up and use idmap (`idmap_pg_dir`
      or `init_idmap_pg_dir`) first
    - idmap covers at least code in `.idmap.text` section
- kernel va (bits 48..63 are 1) is mapped by `TTBR1_EL1`
  - the page table is `swapper_pg_dir` (or `init_pg_dir` when the boot cpu
    starts)
  - there are several regions, `Documentation/arm64/memory.rst`
    - note that `VMALLOC_START` and `KIMAGE_VADDR` overlap
    - when `map_kernel` initializes `swapper_pg_dir`, it calls
      `vm_area_add_early` to add kernel mapping to vmalloc area as well
  - pages that are managed by the buddy allocator are linearly mapped
    - the region is `PAGE_OFFSET` and `PAGE_END`
  - pages that are not (kernel image itself, vmalloc, etc.) are mapped
    differently
- `memstart_addr` is the pa of the start of the usable memory
- addr types
  - va can be stored in `void *`
  - pa can be stored in `phys_addr_t`
  - `__va` and `__pa` convert between va and pa
  - `pfn_to_virt` and `virt_to_pfn` convert between va and pfn
    - pfn is `pa >> PAGE_SHIFT`
- `struct page`
  - how the the buddy allocator manages memory
  - `page_to_phys` and `phys_to_page` depend on the memory model
    - arm64 uses `CONFIG_SPARSEMEM_VMEMMAP` and allows fast conversions

## Exception Levels

- exception levels
  - EL0: userspace
  - EL1: kernel space
  - EL2: hypervisor
  - EL3: firmware / secure monitor
- kernel is designed to run in EL1
- how about KVM?
  - ARMv8.1 VHE allows running unmodified kernel in EL2
  - the host kernel runs in EL2 and the guest kernel runs in EL1
- stack pointer selection
  - each ELn has its own stack pointer `SP_ELn`
  - EL0 can only use `SP_EL0`
  - others can use their `SP_ELn` (ELnh) or use `SP_EL0` (ELnt)

## Exception Vector Table

- exception vectors are defined with `kernel_ventry` in `kernel/entry.S`
  - 64-bit userspace jumps to `el0t_64_*` (defined by `entry_handler`) and
    then to `el0t_64_*_handler`
  - 32-bit userspace jumps to `el0t_32_*` and then to `el0t_32_*_handler`
  - kernel space jumps to `el1h_64_*` and then to `el1h_64_*_handler`
  - `el1t_64_*_handler` are defined by `UNHANDLED` and always panics
- error handler
  - this is NMI, and if fatal, it panics
- fiq handler
  - fast interrupt request
  - this is mostly unused because platforms route this to EL3
    - the only exception is apple m1
- irq handler
  - this is connected to GIC irqchip
  - `gic_handle_irq` gets the irq number and calls `generic_handle_domain_irq`
    - this ultimately calls the generic `handle_fasteoi_irq`
- sync handler
  - this is synchronous because it is generated by cpu rather than externally
  - `svc` (supervisor call) generates an exception and is used for syscalls
  - `sys` generates an exception and is used for cache/tlb maintenance
  - DABT (data abort), IABT (instruction abort)
  - FP/ASIMD (floating point), SVE (Scalable Vector Extension), SME (Scalable
    Matrix Extension)
  - and various exceptions causing the userspace to be killed by `SIGILL`

## CPU Info

- `cpuinfo_store_boot_cpu` and `cpuinfo_store_cpu`
  - they read various msr regs
- when a cpu onlines, `cpuid_cpu_online` is called
  - midr and revid are available at
    `/sys/devices/system/cpu*/regs/identification`
- `/proc/cpuinfo` uses `cpuinfo_op` to show midr
  - bit 0..3: Revision
  - bit 4..15: PartNum
    - 0xd04 is A53
    - 0xd05 is A55
    - 0xd08 is A72
    - 0xd09 is A73
    - 0xd0a is A75
    - 0xd0b is A76
    - 0xd0d is A77
    - 0xd41 is A78
    - 0xd44 is X1
    - 0xd46 is A510
    - 0xd47 is A710
    - 0xd48 is X2
    - 0xd4d is A715
    - 0xd4e is X3
    - 0xd80 is A520
    - 0xd81 is A720
    - 0xd82 is X4
    - 0xd85 is X925
    - 0xd87 is A725
    - `lscpu` can parse this and more
  - bit 16..19: Architecture
    - 0xf means to look elsewhere
  - bit 20..23: Variant
  - bit 24..31: Implementer
    - 0x41 is arm
    - 0x4e is nvidia
    - 0x51 is qualcomm

## Generic Timer

- <https://developer.arm.com/documentation/102379/0101/>
  - Learn the architecture - Generic Timer
  - Processing Element (PE) is a generic term for an implementation of the Arm
    architecture. You can think of a PE as anything that has its own program
    counter and can execute a program.
- on arm, there is the generic timer that consists of a system counter and
  per-core timers
- system counter
  - `CNTFREQ` is the system counter frequency in hz
  - `CNTVCT_EL0` is the current value
- the driver is `CONFIG_ARM_ARCH_TIMER`
  - it relies on arch-specific `include/asm/arch_timer.h`
  - `arch_timer_of_init` registers a `struct clocksource` for the system
    counter
    - the frequency is from `CNTFREQ`
    - as cpus are brought up, it also registers a `struct clock_event_device`
      for each per-core timer
      - the type is `ARCH_TIMER_TYPE_CP15`
  - it appears that the generic timer can also be accessed via mmio
    - `arch_timer_mem_of_init`

## Generic Interrupt Controller

- <https://developer.arm.com/documentation/198123/0302/>
  - Learn the architecture - Arm Generic Interrupt Controller v3 and v4
- on arm, there is the generic interrupt controller
- the drivers are `CONFIG_ARM_GIC_V3` and `CONFIG_ARM_GIC_V3_ITS`
  - ITS, Interrupt Translation Services, can route MSI to cpus
  - `irq_domain_create_tree` creates an `irq_domain`
  - `set_handle_irq` makes `gic_handle_irq` the root irq handler
  - `irq_domain_set_info` sets `irq_desc::handle_irq` to one of
    - `handle_percpu_devid_irq` for SGI/PPI
    - `handle_fasteoi_irq` for SPI
    - `handle_fasteoi_irq` for LPI
- interrupt handling
  - irq exception jumps to different handlers depending on where the cpu is
    running in EL0 or EL1
  - `el0t_64_irq_handler`
    - `enter_from_user_mode`
    - `irq_enter_rcu`
    - `do_interrupt_handler`
    - `irq_exit_rcu`
    - `exit_to_user_mode`
  - `el1h_64_irq_handler`
    - `enter_from_kernel_mode`
    - `irq_enter_rcu`
    - `do_interrupt_handler`
    - `arm64_preempt_schedule_irq`
    - `irq_exit_rcu`
    - `exit_to_kernel_mode`
  - `do_interrupt_handler`
    - calls `gic_handle_irq` on irq stack (use `call_on_irq_stack` if it is on
      the kernel stack)
    - this is done because the kernel stack might not have enough space
- `gic_handle_irq` gets the hw irqnum and calls `__gic_handle_irq`
  - it uses `generic_handle_domain_irq` to map hw irq to `irq_desc` and call
    `desc->handle_irq`

## Context Switches

- current task
  - `current` is defined to `get_current` which reads the current task pointer
    from `SP_EL0`
  - `SP_EL0` is initialized in cpu bringup
    - `init_cpu_task` macro in `kernel/head.S` does the job
    - for the boot cpu, `__primary_switched` initializes it to `init_task`
      - `init_task` is defined in `init/init_task.c` and is static
    - for secondary cpus, `__secondary_switched` initializes it to
      `secondary_data.task`
      - `__cpu_up` sets `secondary_data.task` to the per-cpu idle thread
- task switch
  - `__switch_to` switches from `prev` to `next`
  - `entry_task_switch` saves `next` in per-cpu `__entry_task`
  - `cpu_switch_to`
    - saves the registers of `prev` in `prev->thread.cpu_context`
      - this includes `sp` and `lr`
      - `lr` is the link register and stores the return address of a
        subroutine call
    - retores the registers of `next` from `next->thread.cpu_context`
    - sets `SP_EL0` to `next`
- new task
  - other than `init_task`, all tasks are created by `copy_process`
  - `copy_process` calls the arch-specific `copy_thread`
    - `copy_thread` sets `pc` to `ret_from_fork`
    - when `cpu_switch_to` switches to the new task, it will return to
      `ret_from_fork`
  - `ret_from_fork`
    - calls `schedule_tail` to set up the fresh new task
    - if `x19` is set, this is a kernel thread and `x19` is the starting
      function
      - this is set up in `copy_thread`
    - otherwise, this is a userspace task
      - `get_current_task` reads `SP_EL0`
        - `tsk` is `x28`, defined in `entry.S`
      - `ret_to_user` restores the userspace context and returns to it
- context switch
  - `kernel_exit` macro is used in various return paths
    - after switching to a kernel task or handling an EL1 exception,
      `ret_to_kernel` returns back to the kernel task
    - after switching to a userspace task or handling an EL0 exception,
      `ret_to_user` returns back to the user task
  - `kernel_entry` macro is used in exception vectors
    - EL0 and EL1 exceptions both hit `kernel_entry`
  - `kernel_entry` from EL0 (userspace)
    - `SP_EL0` have been clobbered to point to userspace stack
    - it is restored from `__entry_task`, which was set by `__switch_to`

## Secondary CPUs

- with spin table approach, the secondary cpus loop inside
  `secondary_holding_pen` on boot
  - they wait for `secondary_holding_pen_release` to jump to
    `secondary_startup`
  - when the boot cpu calls `__cpu_up`, `smp_spin_table_cpu_boot` is called to
    release secondary cpus
- `secondary_startup`
  - it enables mmu and jumps to `__secondary_switched`
  - `init_cpu_task` initializes `SP_EL0` (what `get_current` returns) to
    `secondary_data.task`
    - the boot cpu has initialized `secondary_data.task` to the per-cpu idle
      thread in `__cpu_up`
  - `secondary_start_kernel` does post-boot setup and calls
    `cpu_startup_entry` to enter idle

## Flattened Device Tree (FDT)

- `setup_machine_fdt` passes dtb addr to `setup_machine_fdt`
  - `early_init_dt_scan_chosen`
    - `linux,initrd-start`/`linux,initrd-end` for the physical addresses of
      initramfs
    - `bootargs` for kernel cmdline
  - `early_init_dt_scan_root`
  - `early_init_dt_scan_memory`
- `early_init_fdt_scan_reserved_mem` scans `/memreserve/` header or
  `reserved-memory` property and reserves them from memblock
  - `fdt_init_reserved_mem` makes reserved memory available to drivers
  - for example, `rmem_cma_setup` is called for memory reserved for
    `shared-dma-pool`
