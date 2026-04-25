# Kernel Generic Entry

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
    - `secondary_startup_64`
      - `movq current_task(%rdx), %rax` reads per-cpu `current_task`, which is
        the per-cpu idle task
      - `movq TASK_threadsp(%rax), %rsp` restores `esp` from
        `idle->thread.sp`, which is top of `idle->stock`
      - it calls `initial_code`, which is `start_secondary`

## Exception Entries

- arm64
  - `vectors` is the exception vector
  - `kernel_ventry` macro defines an entry snippet
    - it saves source regs to stack
    - it jumps to
      - when from kernel space
        - `el1h_64_sync_handler`
        - `el1h_64_irq_handler`
          - `el1_interrupt` handles irqs
        - `el1h_64_fiq_handler`
        - `el1h_64_error_handler`
      - when from user space
        - `el0t_64_sync_handler`
          - `el0_svc` handles syscalls
        - `el0t_64_irq_handler`
          - `el0_interrupt` handles irqs
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
      - if from userspace, keep using userspace stack
      - if from hardirq contxt (e.g., this is nmi), keep using hardirq context
      - else, switch from per-task kernel stack to per-cpu irq stack to avoid
        kernel stack overflow
    - `irqentry_exit`
  - syscall does not use idt and `entry_SYSCALL_64` is the entry
    - `SWITCH_TO_KERNEL_CR3` switches to kernel pgtable
    - `cpu_current_top_of_stack` is the kernel stack
    - `PUSH_AND_CLEAR_REGS` pushes `pt_regs` to kernel stack
    - it calls `do_syscall_64`
    - `POP_REGS` pops `pt_regs`
    - cleanup and return

## Context Switches

- almost unrelated to generic entry except for `ret_from_fork`
- arm64
  - `__switch_to` performs context switch
    - `cpu_switch_to`
      - it saves regs to `prev->thread.cpu_context`
        - including fp (frame pointer, `x29`), sp, and pc (ret addr, `lr`)
      - it restores regs from `next->thread.cpu_context`
        - we are on the new stack now
      - `current` (`sp_el0`) points to `next`
      - `ret` returns to `next->thread.cpu_context.pc`
        - most likely `__switch_to`
        - but for a brand new task, `ret_from_fork`
  - `copy_thread` inits the new `p->thread`
    - it inits `p->thread.cpu_context` such that `cpu_switch_to` can restore
      from it
    - if not kthread, it copies `pt_regs` to top of stack
    - `p->thread.cpu_context.pc` is `ret_from_fork`
    - `p->thread.cpu_context.sp` is top of stack
  - `ret_from_fork` is the entrypoint of a new task after context switch
    - `x19` and `x20` are kthread func and arg
    - if kthread, it calls the kthread func (which almost never returns)
    - `get_current_task tsk` saves `current` in `x28`
      - `tsk .req x28` aliases `tsk` to `x28`
      - entry code exclusively uses `x28` for `current`
    - `asm_exit_to_user_mode` preps for exit to user
    - `ret_to_user` returns to userspace
      - it restores regs from `pt_regs`
        - `sp_el0` is for userspace stack except kernel uses it for `current`
- x86
  - `__switch_to_asm` performs context switch
    - it builds an `inactive_task_frame` on `prev->stack`
      - the call to `__switch_to_asm` has pushed `ret_addr` on `prev->stack`
    - it saves `rsp` to `prev->thread.sp`
    - it restores `rsp` from `next->thread.sp`
    - it restores regs from `inactive_task_frame` on `next->stack`
      - except for `ret_addr` which is still on `next->stack`
    - it jumps to `__switch_to`
    - when `__switch_to` returns, it pops and jumps to `ret_addr`
  - `copy_thread` synthesizes `inactive_task_frame` on `p->stack`
    - because a new task is an inactive task and `__switch_to_asm` expects
      `inactive_task_frame` on the stack of an inactive task
    - `ret_from_fork_asm` is the return addr of the frame
    - `p->thread.sp` points to the frame
  - `ret_from_fork_asm` is the entrypoint of a new task after context switch
    - `ret_from_fork` is called
      - if kthread, it calls the kthread func (which almost never returns)
      - `syscall_enter_from_user_mode`
    - because the new task is forked from an old task, it returns to the same
      point as the old task

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
