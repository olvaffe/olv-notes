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

## ARM MM

- Kconfig
  - `FLATMEM`
- In `setup_arch`,
  - `paging_init` sets up the page tables
    - `map_lowmem` creates the linear map

## ARM64 MM

- Kconfig
  - `SPARSEMEM_VMEMMAP`
- In `setup_arch`,
  - `paging_init` sets up the page tables

## Booting (Raspberry Pi)

- `kernel/head.S`
  - `_head` defined in `head.S` defines the 64-byte image header
  - the first instruction is `b primary_entry`
  - after some setup and enabling MMU, it jumps to `start_kernel`
  - dtb address is saved to `__fdt_pointer`
- `start_kernel`
  - dmesg `Booting Linux on physical CPU ...`
    - `smp_setup_processor_id`
  - dmesg `Linux version ...`
    - `linux_banner`
  - `setup_arch` is arch-specific
    - dmesg `Machine model: ...`
      - `setup_machine_fdt`
    - dmesg `Reserved memory: create CMA memory pool at ...` and
      `OF: reserved mem:...`
      - `arm64_memblock_init`
      - `early_init_fdt_scan_reserved_mem`
      - `__reserved_mem_init_node`
      - `rmem_cma_setup`
    - dmesg `Zone ranges:` to `%s zone: %lu pages`
      - `bootmem_init`,
      - `zone_sizes_init`
      - `free_area_init`
    - DEBUG dmesg `cpu logical map 0x%llx`
      - `smp_init_cpus`
      - `of_parse_and_init_cpus`
    - DEBUG dmesg `mask of set bits 0x3` and
      `MPIDR hash: aff0[0] aff1[6] aff2[14] aff3[30] mask[0x3] bits[2]`
      - `smp_build_mpidr_hash`
  - dmesg `percpu: Embedded 30 pages/cpu s82008 r8192 d32680 u122880` and
    `pcpu-alloc: s82008 r8192 d32680 u122880 alloc=30*4096`
    - `setup_per_cpu_areas`
  - dmesg `Detected PIPT I-cache on CPU0`
    - `smp_prepare_boot_cpu`
    - `cpuinfo_store_boot_cpu`
    - `__cpuinfo_store_cpu`
    - `cpuinfo_detect_icache_policy`
  - dmesg `CPU features: detected: Spectre-v2` and others
    - `smp_prepare_boot_cpu`
    - `cpuinfo_store_boot_cpu`
    - `init_cpu_features`
    - `setup_boot_cpu_capabilities`
    - `update_cpu_capabilities`
  - dmesg `Built 1 zonelists, mobility grouping on.  Total pages: 996912`
    - `build_all_zonelists`
  - dmesg `Kernel command line:`
    - `saved_command_line`
  - dmesg `Dentry cache hash table entries:` and `Inode-cache hash table entries:`
    - `vfs_caches_init_early`
  - `mm_init`
    - dmesg `mem auto-init: stack:off, heap alloc:off, heap free:off`
      - `report_meminit`
    - dmesg `software IO TLB: mapped [mem ...-...] (64MB)`
      - `mem_init`
      - `swiotlb_init`
      - `swiotlb_init_with_tbl`
      - `swiotlb_print_info`
    - dmesg `Memory: 3818092K/4050944K available (10494K kernel code, 1698K rwdata, 2384K rodata, 5888K init, 587K bss, 167316K reserved, 65536K cma-reserved`
      - `mem_init`
      - `mem_init_print_info`
    - dmesg `SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1`
      - `kmem_cache_init`
  - dmesg `ftrace: allocating ...`
    - `ftrace_init`
  - dmesg `rcu: Hierarchical RCU implementation.` and others
    - `rcu_init`
    - `rcu_bootup_announce`
  - dmesg `NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0`
    - `early_irq_init`
  - dmesg `GIC: Using split EOI/Deactivate mode`
    - `init_IRQ`
    - `irqchip_init`
    - `of_irq_init`
    - `gic_of_init`
    - `__gic_init_bases`
  - dmesg `random: get_random_bytes called from ...`
  - dmesg `arch_timer: cp15 timer(s) running at 54.00MHz (phys).`
    - `time_init`
    - `timer_probe`
    - `arch_timer_of_init`
    - `arch_timer_common_init`
    - `arch_timer_banner`
  - dmesg `clocksource: arch_sys_counter: mask: 0xffffffffffffff ...`
    - `time_init`
    - `timer_probe`
    - `arch_timer_of_init`
    - `arch_timer_common_init`
    - `arch_counter_register`
    - `clocksource_register_hz`
    - `__clocksource_register_scale`
    - `__clocksource_update_freq_scale`
  - dmesg `sched_clock: 56 bits at 54MHz, resolution 18ns, wraps every 4398046511102ns`
    - `time_init`
    - `timer_probe`
    - `arch_timer_of_init`
    - `arch_timer_common_init`
    - `arch_counter_register`
    - `sched_clock_register`
  - dmesg `Console: colour dummy device 80x25` and `printk: console [tty0] enabled`
    - `console_init`
    - `con_init`
  - dmesg `Calibrating delay loop (skipped), value calculated using timer frequency.. 108.50 BogoMIPS ...`
    - `calibrate_delay`
  - dmesg `pid_max: default: 32768 minimum: 301`
    - `pid_idr_init`
  - dmesg `Mount-cache hash table entries: ...` and `Mountpoint-cache hash table entries: ...`
    - `vfs_caches_init`
    - `mnt_init`
  - `arch_call_rest_init`
    - `rest_init`
    - spawns a thread to run `kernel_init` and goes into idle
- `kernel_init_freeable` called by `kernel_init`
  - dmesg `CPU0: cluster 0 core 0 thread -1 mpidr 0x00000080000000`
    - `smp_prepare_cpus`
    - `store_cpu_topology`
  - `do_pre_smp_initcalls`
    - call all `early_initcall()`, including
    - `trace_init_flags_sys_enter`
    - `trace_init_flags_sys_exit`
    - `cpu_suspend_init`
    - `asids_init`
    - `spawn_ksoftirqd`
    - `migration_init`
    - `srcu_bootup_announce`
    - `rcu_spawn_core_kthreads`
    - `rcu_spawn_gp_kthread`
    - `check_cpu_stall_init`
    - `rcu_sysrq_init`
    - `cpu_stop_init`
    - `init_events`
    - `init_trace_printk`
    - `event_trace_enable_again`
    - `initialize_ptr_random`
    - `its_pmsi_init`
    - `its_pci_msi_init`
    - `dummy_timer_register`
  - `smp_init`
    - dmesg `smp: Bringing up secondary CPUs ...`
    - dmesg `Detected PIPT I-cache on CPU1`
        - the boot CPU
          - `bringup_nonboot_cpus`
          - `cpu_up`
          - `_cpu_up`
          - `cpuhp_up_callbacks`
          - `cpuhp_invoke_callback`
          - `bringup_cpu`
          - `__cpu_up`
          - `boot_secondary`
          - `smp_spin_table_cpu_boot`
          - `write_pen_release`
        - the secondary CPUs
          - `secondary_holding_pen`
          - `secondary_startup`
          - `__secondary_switched`
          - `secondary_start_kernel`
          - `cpuinfo_store_cpu`
            - `__cpuinfo_store_cpu`
            - `cpuinfo_detect_icache_policy`
          - dmesg `CPU1: cluster 0 core 1 thread -1 mpidr 0x00000080000001`
            - `secondary_start_kernel`
            - `store_cpu_topology`
          - dmesg `CPU1: Booted secondary processor 0x0000000001 [0x410fd083]`
            - `secondary_start_kernel`
    - dmesg `smp: Brought up 1 node, 4 CPUs`
    - `smp_cpus_done`
      - dmesg `SMP: Total of 4 processors activated.`
      - dmesg `CPU features: detected: 32-bit EL0 Support` and `CPU features: detected: CRC32 instructions`
        - `setup_cpu_features`
        - `setup_system_capabilities`
        - `update_cpu_capabilities`
      - dmesg `CPU: All CPU(s) started at EL2`
        - `hyp_mode_check`
      - dmesg `alternatives: patching kernel code`
        - `apply_alternatives_all`
        - `__apply_alternatives_multi_stop`
        - `__apply_alternatives`
  - dmesg `devtmpfs: initialized`
    - `do_basic_setup`
    - `driver_init`
    - `devtmpfs_init`
- `do_initcalls` called by `do_basic_setup`
  - `pure_initcall()`s
    - `ipc_ns_init`
    - `init_mmap_min_addr`
    - `pci_realloc_setup_params`
    - `net_ns_init`
  - `core_initcall()`s
    - `fpsimd_init`
    - `tagged_addr_init`
    - `enable_mrs_emulation`
    - `armv8_deprecated_init`
    - `map_entry_trampoline`
    - `alloc_frozen_cpus`
    - `cpu_hotplug_pm_sync_init`
    - `wq_sysfs_init`
    - `ksysfs_init`
    - `schedutil_gov_init`
    - `pm_init`
    - `rcu_set_runtime_mode`
    - `rcu_spawn_tasks_rude_kthread`
    - `dma_init_reserved_memory`
    - `init_jiffies_clocksource`
    - `futex_init`
    - `cgroup_wq_init`
    - `cgroup1_wq_init`
    - `ftrace_mod_cmd_init`
    - `init_graph_trace`
    - `cpu_pm_init`
    - `init_zero_pfn`
    - `cma_init_reserved_areas`
    - `fsnotify_init`
    - `filelock_init`
    - `init_script_binfmt`
    - `init_elf_binfmt`
    - `init_compat_elf_binfmt`
    - `debugfs_init`
    - `tracefs_init`
    - `register_xor_blocks`
    - `prandom_init_early`
    - `pinctrl_init`
    - `gpiolib_dev_init`
    - `regulator_init`
    - `iommu_init`
    - `component_debug_init`
    - `genpd_bus_init`
    - `soc_bus_register`
    - `register_cpufreq_notifier`
    - `opp_debug_init`
    - `cpufreq_core_init`
    - `cpufreq_gov_performance_init`
    - `cpufreq_dt_platdev_init`
    - `cpuidle_init`
    - `sock_init`
    - `net_inuse_init`
    - `net_defaults_init`
    - `init_default_flow_dissectors`
    - `netpoll_init`
    - `netlink_proto_init`
    - `genl_init`
  - `postcore_initcall()`s
    - `debug_monitors_init`
    - `irq_sysfs_init`
    - `dma_atomic_pool_init`
    - `release_early_probes`
    - `bdi_class_init`
    - `mm_sysfs_init`
    - `init_per_zone_wmark_min`
    - `mpi_init`
    - `kobject_uevent_init`
    - `pcibus_class_init`
    - `pci_driver_init`
    - `amba_init`
    - `tty_class_init`
    - `vtconsole_class_init`
    - `serdev_init`
    - `iommu_dev_init`
    - `mipi_dsi_bus_init`
    - `devlink_class_init`
    - `software_node_init`
    - `wakeup_sources_debugfs_init`
    - `wakeup_sources_sysfs_init`
    - `regmap_initcall`
    - `spi_init`
    - `i2c_init`
    - `thermal_init`
    - `init_menu`
  - `arch_initcall()`s
    - `reserve_memblock_reserved_regions`
    - `aarch32_alloc_vdso_pages`
    - `vdso_init`
    - `arch_hw_breakpoint_init`
    - `asids_update_limit`
    - `cryptomgr_init`
    - `dma_channel_table_init`
    - `dma_bus_init`
    - `iommu_dma_init`
    - `of_platform_default_populate_init`
  - `subsys_initcall()`s
    - `topology_init`
    - `uid_cache_init`
    - `param_sysfs_init`
    - `user_namespace_sysctl_init`
    - `time_ns_init`
    - `cgroup_sysfs_init`
    - `cgroup_namespaces_init`
    - `user_namespaces_init`
    - `oom_init`
    - `default_bdi_init`
    - `percpu_enable_async`
    - `kcompactd_init`
    - `init_user_reserve`
    - `init_admin_reserve`
    - `init_reserve_notifier`
    - `swap_init_sysfs`
    - `swapfile_init`
    - `hugepage_init`
    - `io_wq_init`
    - `rsa_init`
    - `crypto_cmac_module_init`
    - `hmac_module_init`
    - `crypto_null_mod_init`
    - `sha256_generic_mod_init`
    - `blake2b_mod_init`
    - `crypto_ecb_module_init`
    - `crypto_ctr_module_init`
    - `crypto_gcm_module_init`
    - `crypto_ccm_module_init`
    - `aes_init`
    - `crc32c_mod_init`
    - `xxhash_mod_init`
    - `drbg_init`
    - `ghash_mod_init`
    - `ecdh_init`
    - `init_bio`
    - `blk_settings_init`
    - `blk_ioc_init`
    - `blk_mq_init`
    - `genhd_device_init`
    - `raid6_select_algo`
    - `gpiolib_debugfs_init`
    - `pwm_debugfs_init`
    - `pwm_sysfs_init`
    - `pci_slot_init`
    - `fbmem_init`
    - `regulator_fixed_voltage_init`
    - `gpio_regulator_init`
    - `misc_init`
    - `iommu_subsys_init`
    - `vga_arb_device_init`
    - `register_cpu_capacity_sysctl`
    - `dma_buf_init`
    - `phy_init`
    - `usb_common_init`
    - `usb_init`
    - `serio_init`
    - `input_init`
    - `rtc_init`
    - `power_supply_class_init`
    - `hwmon_init`
    - `mmc_init`
    - `leds_init`
    - `arm_pmu_hp_init`
    - `nvmem_init`
    - `init_soundcore`
    - `alsa_sound_init`
    - `proto_init`
    - `net_dev_init`
    - `neigh_init`
    - `fib_notifier_init`
    - `ethnl_init`
    - `nexthop_init`
    - `bt_init`
    - `ieee80211_init`
    - `rfkill_init`
    - `watchdog_init`
  - `fs_initcall()`s
    - `create_debug_debugfs_entry`
    - `clocksource_done_booting`
    - `tracer_init_tracefs`
    - `init_trace_printk_function_export`
    - `init_graph_tracefs`
    - `init_dynamic_event`
    - `init_uprobe_trace`
    - `init_pipe_fs`
    - `inotify_user_setup`
    - `eventpoll_init`
    - `anon_inode_init`
    - `proc_locks_init`
    - `iomap_init`
    - `proc_cmdline_init`
    - `proc_consoles_init`
    - `proc_cpuinfo_init`
    - `proc_devices_init`
    - `proc_interrupts_init`
    - `proc_loadavg_init`
    - `proc_meminfo_init`
    - `proc_stat_init`
    - `proc_uptime_init`
    - `proc_version_init`
    - `proc_softirqs_init`
    - `proc_kmsg_init`
    - `proc_page_init`
    - `init_ramfs_fs`
    - `blk_scsi_ioctl_init`
    - `dynamic_debug_init_control`
    - `simplefb_init`
    - `chr_dev_init`
    - `firmware_class_init`
    - `sysctl_core_init`
    - `eth_offload_init`
    - `ipv4_offload_init`
    - `inet_init`
    - `af_unix_init`
    - `ipv6_offload_init`
    - `cfg80211_init`
    - `pci_apply_final_quirks`
  - `rootfs_initcall()`s
    - `populate_rootfs`
  - `device_initcall()`s
    - `register_arm64_panic_block`
    - `cpuinfo_regs_init`
    - `armv8_pmu_driver_init`
    - `arch_init_uprobes`
    - `proc_execdomains_init`
    - `register_warn_debugfs`
    - `cpuhp_sysfs_init`
    - `ioresources_init`
    - `init_sched_debug_procfs`
    - `irq_pm_init_ops`
    - `timekeeping_init_ops`
    - `init_clocksource_sysfs`
    - `init_timer_list_procfs`
    - `alarmtimer_init`
    - `init_posix_timers`
    - `clockevents_init_sysfs`
    - `sched_clock_syscore_init`
    - `proc_modules_init`
    - `kallsyms_init`
    - `pid_namespaces_init`
    - `ikconfig_init`
    - `seccomp_sysctl_init`
    - `utsname_sysctl_init`
    - `init_tracepoints`
    - `perf_event_sysfs_init`
    - `system_trusted_keyring_init`
    - `kswapd_init`
    - `extfrag_debug_init`
    - `mm_compute_batch_init`
    - `slab_proc_init`
    - `workingset_init`
    - `proc_vmalloc_init`
    - `memblock_init_debugfs`
    - `procswaps_init`
    - `slab_sysfs_init`
    - `fcntl_init`
    - `proc_filesystems_init`
    - `start_dirtytime_writeback`
    - `blkdev_init`
    - `dio_init`
    - `aio_setup`
    - `io_uring_init`
    - `init_devpts_fs`
    - `init_fat_fs`
    - `init_vfat_fs`
    - `init_exfat_fs`
    - `init_nls_cp437`
    - `init_nls_iso8859_1`
    - `init_nls_utf8`
    - `fuse_init`
    - `ipc_init`
    - `ipc_sysctl_init`
    - `init_mqueue_fs`
    - `key_proc_init`
    - `crypto_algapi_init`
    - `jent_mod_init`
    - `calibrate_xor_blocks`
    - `asymmetric_key_init`
    - `x509_key_init`
    - `proc_genhd_init`
    - `bsg_init`
    - `deadline_init`
    - `kyber_init`
    - `libcrc32c_mod_init`
    - `percpu_counter_startup`
    - `bcm2835_pinctrl_driver_init`
    - `rpi_exp_gpio_driver_init`
    - `bcm2835_pwm_driver_init`
    - `pci_proc_init`
    - `brcm_pcie_driver_init`
    - `of_fixed_factor_clk_driver_init`
    - `of_fixed_clk_driver_init`
    - `gpio_clk_driver_init`
    - `clk_dvp_driver_init`
    - `bcm2835_clk_driver_init`
    - `bcm2835_aux_clk_driver_init`
    - `raspberrypi_clk_driver_init`
    - `bcm2835_dma_driver_init`
    - `bcm2835_power_driver_init`
    - `rpi_power_driver_init`
    - `rpi_reset_driver_init`
    - `reset_simple_driver_init`
    - `n_null_init`
    - `pty_init`
    - `hwrng_modinit`
    - `bcm2835_rng_driver_init`
    - `iproc_rng200_driver_init`
    - `cavium_rng_pf_driver_init`
    - `cavium_rng_vf_driver_init`
    - `drm_kms_helper_init`
    - `drm_core_init`
    - `topology_sysfs_init`
    - `cacheinfo_sysfs_init`
    - `bcm2835_pm_driver_init`
    - `bcm2835_spi_driver_init`
    - `bcm2835aux_spi_driver_init`
    - `net_olddevs_init`
    - `blackhole_netdev_init`
    - `phy_module_init`
    - `phy_module_init`
    - `fixed_mdio_bus_init`
    - `unimac_mdio_driver_init`
    - `bcmgenet_driver_init`
    - `brcmfmac_module_init`
    - `xhci_hcd_init`
    - `xhci_pci_init`
    - `serport_init`
    - `input_leds_init`
    - `evdev_init`
    - `bcm2835_i2c_driver_init`
    - `brcmstb_i2c_driver_init`
    - `rpi_hwmon_driver_init`
    - `bcm2835_thermal_driver_init`
    - `bcm2835_wdt_driver_init`
    - `hci_uart_init`
    - `dt_cpufreq_platdrv_init`
    - `raspberrypi_cpufreq_driver_init`
    - `arm_idle_init`
    - `mmc_pwrseq_simple_driver_init`
    - `mmc_pwrseq_emmc_driver_init`
    - `mmc_blk_init`
    - `sdhci_drv_init`
    - `sdhci_pltfm_drv_init`
    - `sdhci_iproc_driver_init`
    - `rpi_firmware_driver_init`
    - `smccc_soc_init`
    - `hid_init`
    - `hid_generic_init`
    - `a4_driver_init`
    - `apple_driver_init`
    - `belkin_driver_init`
    - `ch_driver_init`
    - `ch_driver_init`
    - `cp_driver_init`
    - `ez_driver_init`
    - `ite_driver_init`
    - `ks_driver_init`
    - `lg_driver_init`
    - `lg_g15_driver_init`
    - `ms_driver_init`
    - `mr_driver_init`
    - `redragon_driver_init`
    - `hid_init`
    - `vchiq_driver_init`
    - `bcm2835_alsa_driver_init`
    - `bcm2835_mbox_driver_init`
    - `alsa_timer_init`
    - `alsa_pcm_init`
    - `snd_soc_init`
    - `sock_diag_init`
    - `gre_offload_init`
    - `sysctl_ipv4_init`
    - `tunnel4_init`
    - `inet_diag_init`
    - `tcp_diag_init`
    - `cubictcp_register`
    - `inet6_init`
    - `sit_init`
    - `packet_init`
  - `late_initcall()`s
    - `init_oops_id`
    - `sched_init_debug`
    - `cpu_latency_qos_init`
    - `pm_debugfs_init`
    - `printk_late_init`
    - `init_srcu_module_notifier`
    - `swiotlb_create_debugfs`
    - `tk_debug_sleep_time_init`
    - `load_system_certificate_list`
    - `fault_around_debugfs`
    - `max_swapfiles_check`
    - `split_huge_pages_debugfs`
    - `check_early_ioremap_leak`
    - `init_btrfs_fs`
    - `init_root_keyring`
    - `blk_timeout_init`
    - `prandom_init_late`
    - `pci_resource_alignment_sysfs_init`
    - `pci_sysfs_init`
    - `amba_deferred_retry`
    - `clk_debug_init`
    - `sync_state_resume_initcall`
    - `deferred_probe_initcall`
    - `genpd_power_off_unused`
    - `genpd_debug_init`
    - `init_netconsole`
    - `of_fdt_raw_init`
    - `tcp_congestion_default`
    - `regulatory_init_db`
    - `init_amu_fie`
    - `clear_boot_tracer`
    - `clk_disable_unused`
    - `regulator_init_complete`
    - `of_platform_sync_state_init`
    - `alsa_sound_last_init`
- `kernel_init`
  - dmesg `Warning: unable to open an initial console.`
    - `kernel_init_freeable`
    - `console_on_rootfs`
  - dmesg `Freeing unused kernel memory: 5888K`
    - `free_initmem`
    - `free_reserved_area`
  - dmesg `Run /init as init process`
    - `run_init_process`

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

## KVM

- ELs
  - traditionally,
    - EL0: userspace
    - EL1: kernel
  - armv7a has optional hyp mode
    - EL0: userspace
    - EL1: most of kernel
    - EL2: kernel when running hyp code
      - this is also known as split-mode virtualization or nVHE (non-VHE)
  - armv8.1 has vhe (Virtualization Host Extension)
    - EL0: userspace
    - EL2: kernel
  - inside the vm,
    - EL0: guest userspace
    - EL1: guest kernel
      - it traps to EL2 in the host
  - kernel also has hVHE mode, which uses VHE but splits the kernel like in
    hyp
    - EL0: userspace
    - EL1: most of kernel
    - EL2: kernel when running hyp code
- instructions
  - `svc` traps from EL0 to EL1, used for syscalls
  - `hvc` traps from EL1+ to EL2, used for hypercalls
  - `smc` traps from EL1+ to EL3, used for secure monitor calls
- boot
  - cpu init
    - the boot cpu starts in `primary_entry`, calls `init_kernel_el` and
      `set_cpu_boot_mode_flag`
    - the secondary cpus start in `secondary_holding_pen` (or elsewhere),
      calls `init_kernel_el` and `set_cpu_boot_mode_flag`
  - `init_kernel_el`
    - if cpu is in EL1,
      - inits `sctlr_el1` to `INIT_SCTLR_EL1_MMU_OFF`
      - inits `spsr_el1` to `INIT_PSTATE_EL1`
      - inits `elr_el1` to the return addr
      - inits `w0` to `BOOT_CPU_MODE_EL1`
      - `eret` returns to `elr_el1` at the EL specified by `spsr_el1`, which
        is EL1
    - if cpu is in EL2,
      - inits `elr_el2` to the return addr
      - inits `sctlr_el2` to `INIT_SCTLR_EL2_MMU_OFF`
      - inits `hcr_el2` to `#HCR_E2H`
        - this attemps to enable VHE, which allows the kernel to run at EL2
      - inits `vbar_el2` to `__hyp_stub_vectors`
      - reads `hcr_el2` back to check for VHE support
      - inits `sctlr_el1` to `INIT_SCTLR_EL1_MMU_OFF`
      - `__init_el2_nvhe_prepare_eret` inits `spsr_el2` to `INIT_PSTATE_EL1`
      - inits `w0` to `BOOT_CPU_MODE_EL1`
        - if VHE, `BOOT_CPU_MODE_EL2` too
      - `eret` returns to `elr_el2` at the EL specified by `spsr_el2`, which
        is EL1
  - `set_cpu_boot_mode_flag`
    - `__boot_cpu_mode` is statically initialized to `{EL2, EL1}`
    - when all cpus are in EL2, it is set to `{EL2, EL2}`
    - when all cpus are in EL1, it is set to `{EL1, EL1}`
    - when mixed, it is set to `{EL1, EL2}` which is invalid
  - `finalise_el2`
    - cpu is in EL1
    - `hvc` traps to `elx_sync` in EL2 and calls `__finalise_el2`
    - `eret` stays at EL2 if VHE is supported
- `is_hyp_mode_available` returns true when `__boot_cpu_mode` is EL2
  - both VHE and nVHE boot in EL2
  - KVM is disabled when this returns false
- `is_kernel_in_hyp_mode` returns true if the cpu is currently in EL2
  - this is true when VHE
  - false when nVHE or hVHE (split virt), because most of the kernel runs in
    EL1
- `module_init(kvm_arm_init)`
  - if doing split virt, most of the kernel runs in EL1 and `init_hyp_mode`
    preps EL2 to run hyp code
- `KVM_CREATE_VM` ioctl
  - `kvm_arch_init_vm`
    - `kvm->arch` has arch-specific type `kvm_arch`
    - `kvm_share_hyp` shares the `kvm` struct with EL2 address space
      - nop if already in EL2
    - `kvm_init_stage2_mmu` inits `kvm->arch.mmu`
      - `kvm_pgtable_stage2_init`
  - `kvm_arch_hardware_enable`
    - if split virt,
      - `hyp_install_host_vector` installs `__kvm_hyp_init` vector for EL2
      - `arm_smccc_1_1_hvc(...)` traps to `__do_hyp_init` in EL2 and calls
        `___kvm_hyp_init`
        - this installs `__kvm_hyp_host_vector` vector for EL2
      - `cpu_set_hyp_vector` installs `__kvm_hyp_vector` vector for EL2
  - `kvm_arch_post_init_vm`
- `KVM_CREATE_VCPU` ioctl
  - `kvm_arch_vcpu_precreate`
  - `kvm_arch_vcpu_create`
  - `kvm_arch_vcpu_postcreate`
- `KVM_ARM_VCPU_INIT` ioctl
- `KVM_SET_USER_MEMORY_REGION2` ioctl
  - `kvm_arch_prepare_memory_region`
  - `kvm_arch_commit_memory_region`
  - when guest accesses an addr whose stage-2 mapping hasn't been setup,
    `kvm_handle_guest_abort` is called
    - `kvm_pgtable_stage2_map` sets up stage-2 mapping
- `KVM_RUN` ioctl
  - `kvm_arch_vcpu_ioctl_run` 
    - `kvm_arm_vcpu_enter_exit` calls `__kvm_vcpu_run`
      - two variants depending on whether VHE or nVHE/hVHE
