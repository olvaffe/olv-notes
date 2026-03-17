Kernel Generic Entry
====================

## CPU Entries

- arm64
  - `vectors` is the exception vector
  - `kernel_ventry` macro defines an entry snippet
    - it saves source regs to stack
    - it jumps to
      - when from kernel space
        - `el1h_64_sync_handler`
        - `el1h_64_irq_handler`
        - `el1h_64_fiq_handler`
        - `el1h_64_error_handler`
      - when from user space
        - `el0t_64_sync_handler`
        - `el0t_64_irq_handler`
        - `el0t_64_fiq_handler`
        - `el0t_64_error_handler`
      - etc
- x86
  - `def_idts` is the idt table
  - `DECLARE_IDTENTRY` assembly macro defines an entry
    - `error_entry`
      - `PUSH_AND_CLEAR_REGS` pushes `pt_regs` to temp stack
      - `SWITCH_TO_KERNEL_CR3` switches to kernel pgtable
      - `sync_regs` copies `pt_regs` to kernel stack
    - `movq %rax, %rsp` switches to kernel stack
    - it calls one of
      - `exc_page_fault` wrapper
      - `common_interrupt` wrapper
      - ipi, etc.
    - cleanup and return
  - `DEFINE_IDTENTRY` C macro defines a C wrapper for most exceptions
    - `irqentry_enter`
    - real func
    - `irqentry_exit`
  - `DECLARE_IDTENTRY_IRQ` C macro defines a C wrapper for external irq
    - `irqentry_enter`
    - `run_irq_on_irqstack_cond(real func)`
    - `irqentry_exit`
  - syscall does not use idt and `entry_SYSCALL_64` is the entry
    - `SWITCH_TO_KERNEL_CR3` switches to kernel pgtable
    - `cpu_current_top_of_stack` is the kernel stack
    - `PUSH_AND_CLEAR_REGS` pushes `pt_regs` to kernel stack
    - it calls `do_syscall_64`
    - `POP_REGS` pops `pt_regs`
    - cleanup and return

## Boot Entries

- unrelated to generic entry
- uefi jumps to `AddressOfEntryPoint` in PE header, an addr relative to start
  of the image
  - arm64: `efi_pe_entry`
    - the field is `__efistub_efi_pe_entry - .L_head`
      - `.L_head` is the start
      - `__efistub_efi_pe_entry` is `efi_pe_entry` because libstub is built with
        `--prefix-symbols=__efistub_`
    - `efi_pe_entry -> efi_stub_common -> efi_boot_kernel -> efi_enter_kernel -> primary_entry`
  - x86: `efi_pe_entry`
    - the field is `ZO_efi_pe_entry`
      - zimage offset of `efi_pe_entry`, defined in generated `zoffset.h`
    - `efi_pe_entry -> efi_stub_entry -> (decompression) -> enter_kernel -> startup_64`
  - `CONFIG_EFI_ZBOOT`: `efi_zboot_entry`
    - `efi_zboot_entry -> (decompression) -> efi_stub_common -> ...`
- arm64
  - current task and kernel stack
    - current task is stored in `sp_el0`
    - kernel stack is at `current->stack`
  - `primary_entry` is the entrypoint for boot cpu
    - if not using uefi, bootloader jumps to the start of the image which has
      `nop; b primary_entry`
    - `__primary_switched`
      - `init_cpu_task` inits current stack and kernel stack to `&init_task`
      - it uses `vectors` as the exception vector
      - it jumps to `start_kernel`
  - `secondary_entry` is the entrypoint for secondary cpus
    - primary cpu calls `__cpu_up` to boot secondary cpu
      - `secondary_data.task` points to the per-cpu idle task
      - `boot_secondary` calls `cpu_psci_cpu_boot` to boot secondary cpu from
        `secondary_entry`
    - `__secondary_switched`
      - it uses `vectors` as the exception vector
      - `init_cpu_task` init current task and kernel stack to `secondary_data.task`
      - it jumps to `secondary_start_kernel`
- x86
  - current task and kernel stack
    - current task is stored in per-cpu `current_task`
    - kernel stack is at `current->stack`
    - `esp` is saved in `current->thread.sp`
  - `startup_64` is the entrypoint for boot cpu
    - it jumps to `common_startup_64` after init
    - `movq current_task(%rdx), %rax` reads per-cpu `current_task`, which is
      statically initialized to `&init_task`
    - `movq TASK_threadsp(%rax), %rsp` restores `esp` from
      `init_task.thread.sp`, which is statically initialized to
      `__top_init_kernel_stack`
    - it calls `initial_code`, which is statically initialized to
      `x86_64_start_kernel`
  - `trampoline_start` is the entrypoint for secondary cpus
    - primary cpu calls `arch_cpuhp_kick_ap_alive` to boot secondary cpu
      - `common_cpu_up` points per-cpu `current_task` to the per-cpu idle task
      - `do_boot_cpu` boots secondary cpu
        - `idle->thread.sp` is set to `task_pt_regs(idle)`, top of
          `idle->stack`
        - `initial_code` is set to `start_secondary`
    - `trampoline_start` transitions to protected mode and jumps to `startup_32`
    - `startup_32` transitions to 64-bit mode and jumps to `startup_64`
    - `startup_64` jumps to `secondary_startup_64`
    - `movq current_task(%rdx), %rax` reads per-cpu `current_task`, which is
      the per-cpu idle task
    - `movq TASK_threadsp(%rax), %rsp` restores `esp` from
      `idle->thread.sp`, which is top of `idle->stock`
    - it calls `initial_code`, which is `start_secondary`

## Generic Entries

- syscall
  - `syscall_enter_from_user_mode`
  - `syscall_exit_to_user_mode`
- exceptions
  - `irqentry_enter`
  - if external irq
    - `irq_enter_rcu`
    - `irq_exit_rcu`
  - `irqentry_exit`
