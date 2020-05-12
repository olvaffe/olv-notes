Kernel and IRQ
==============

## Boot Time Setup

- IDT size is 256 with the first 32 reserved for processor exceptions (traps)
  - `IDT_ENTRIES` and `NR_VECTORS` is 256
  - `NUM_EXCEPTION_VECTORS` and `FIRST_EXTERNAL_VECTOR` is 32
  - traps include
    - `X86_TRAP_DE` divide-by-zero
    - `X86_TRAP_NMI` NMI
    - `X86_TRAP_OF` overflow
    - `X86_TRAP_PF` page fault
    - and others
- `NR_IRQS` is `NR_VECTORS + IO_APIC_VECTOR_LIMIT`
  - `FIRST_SYSTEM_VECTOR` is 256, following IDT
  - unless when there is local APIC, then `FIRST_SYSTEM_VECTOR` is 0xec
- In `go_to_protected_mode`,
  - `realmode_switch_hook` issues `cli` to disable local irq
  - `mask_all_interrupts` masks out all irq in PIC
  - loads a null IDT
  - jumps to protect mode and call `startup_32`
    - which jumps to long mode (64-bit) and call `startup_64`
    - which calls `x86_64_start_kernel`
- In `x86_64_start_kernel`,
  - `idt_setup_early_handler` sets up the IDT from `early_idt_handler_array`,
    where all traps are handled by `early_idt_handler_common` that sets
    up early pgtable and works around bugs
  - at the end, `start_kernel` is called
- In `setup_arch` called by `start_kernel`,
  - `idt_setup_early_traps` updates some trap handlers in IDT
  - `idt_setup_early_pf` updated `X86_TRAP_PF` handler
- In `start_kernel` after `setup_arch`,
  - `trap_init` calls `idt_setup_traps` and `idt_setup_ist_traps` to set the
    final trap handlers
  - `early_irq_init` initializes the kernel IRQ subsystem and calls
    `arch_early_irq_init`
  - `init_IRQ` initializes all HW IRQs after traps.
    `idt_setup_apic_and_irq_gates` initializes that part of IDT from
    `irq_entries_start` to be handled by `common_interrupt` and `do_IRQ`
  - `local_irq_enable` is called

## Hardware Interrupts

- PIC maps hardware interrups, IRQ#n, to vectors in IDT 
- when a hardware interrupt is received, CPU disables interrupts (`cli`)
  automatically
- IDT entries from `FIRST_EXTERNAL_VECTOR` to `NR_VECTORS` point to the table
  at `irq_entries_start`.  The entries handle interrupts in a uniform way
  - HW disables interrups and pushes the original context registers onto stack
    before entry
  - `interrupt_entry` switches to the kernel stack
    (`cpu_current_top_of_stack`), pushes the original context registers to the
    kernel stack, construct `pt_regs` on kernel stack, switches to the irq
    stack (`hardirq_stack_ptr`)
  - call `do_IRQ`
- IRQ handling
  - `early_irq_init` sets up irq to `irq_desc` mappings
  - `idt_setup_apic_and_irq_gates` sets up IDT
  - `irq_set_chip_and_handler` updates an `irq_desc` to point to the specified
    `irq_chip` and handler
  - when a hw irq happens, `IDT -> do_IRQ -> irq_desc::handle_irq`
  - the handler acks the irq, and calls each action in the `irq_desc::action`
    list
  - `request_irq` registers an action to an `irq_desc`

## Syscalls

- x86-64 has an instruction, `syscall`, that loads rip from `MSR_LSTAR` MSR.
  - the intr also loads cs/ss from `MSR_STAR` MSR
  - `syscall_init` initializes `MSR_LSTAR` to `entry_SYSCALL_64` and
    `MSR_STAR` to `__KERNEL_CS`
- `entry_SYSCALL_64` does
  - save user rsp and set it to `cpu_current_top_of_stack`
  - construct `pt_regs` on stack
  - call `do_syscall_64`
    - enable IRQ for kernel mode
    - execute the syscall
    - disable IRQ for user mode
  - `iretq` or `sysretq` that restore flags

## Privilege

* In an IDT entry, there is a 16-bits segment selector and a 2-bits DPL.
  * The selector decides thew new segment
  * The DPL decides the lowest privilege that can interrupt
  * `cs` gives CPL.  If `CPL >= PL of new segment` and `CPL <= DPL`, interrupt
    succeeds.  The idea is so that the interrupt handler has higher privilege
    than the program caused the interrupt and the program has enough privilege
    to cause the interrupt.
  * `cs:eip` becomes `segment selector:offset` in the IDT entry.
* In `setup_gdt` in `boot/pm.c`, `GDT_ENTRY_BOOT_CS` and `GDT_ENTRY_BOOT_DS`
  GDT have acess flags `0x9b` and `0x93`.  Both indicate highest privilege.

## Preempt count

- By default
  - `PREEMPT_MASK: 0x000000ff`
  - `SOFTIRQ_MASK: 0x0000ff00`
  - `HARDIRQ_MASK: 0x000f0000`
  - `NMI_MASK:     0x00100000`
- `#define in_irq()		(hardirq_count())`
- `#define in_softirq()		(softirq_count())`
- `#define in_interrupt()	(irq_count())`
- `preempt_count()` is per-thread
- `preempt_disable()` increments `preempt_count`
- `preempt_enable()` decrements `preempt_count`; when 0 is reached, call
  `__preempt_schedule`
- `__irq_enter` increments `prement_count` by `HARDIRQ_OFFSET`
- `local_irq_save` calls `arch_local_irq_save`
  - on x86, it `pushf; pop` to saves EFLAGS register to flags and then `cli`
- `local_irq_restore` restores to the saved flags
  - on x86, it `push; popf` to restore EFLAGS register from flags
- `spin_lock` on UP simply calls `preempt_disable`; it does the real stuff
  only on SMP
- `spin_lock_irqsave` on UP simply calls `local_irq_save`; on SMP, it does
  - `local_irq_save`
  - `preempt_disable`
  - real locking
