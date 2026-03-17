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
