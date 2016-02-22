Kernel and IRQ
==============

## Boot Time Setup

* Before going to protected mode in `go_to_protected_mode`, `cli` is issued to
  disable local irq and `mask_all_interrupts` is called to mask all irq in PIC.
* In `kernel/head_32.S`, IDT (interrupt descriptor table) is created.
  * `setup_idt` is called after paging is enabled and stack is set up
  * `idt_table` is defined in `kernel/traps.c`.
  * All 256 entries of IDT are set up to call `ignore_int`, except for several
    of them who point to `early_divide_err`, `early_illegal_opcode`,
    `early_protection_fault`, and `early_page_fault` respectively.
* In `start_kernel`, after `setup_arch` and before `mem_init`,
  * `trap_init` is called
  * `early_irq_init` is called
  * `init_IRQ` is called
  * `early_boot_irqs_on` is called to set `early_boot_irqs_enabled` to true.
  * `local_irq_enable` is called to enable local irq.
* In `kernel/entry_32.S`, an array of function pointers is defined in
  `interrupt`.
  * It has members as many as the number of user-defined vectores.
  * They point to code snippet that pushes the vector number and then indirectly
    jumps to `common_interrupt`.  It plays a trick to make it stack look exactly
    like `pt_regs`, with `orig_ax = ~vector_number`.  It then calls `do_IRQ` and
    pasees its stack as the first argument.
* `init_IRQ` is actually `native_init_IRQ`.
  * vector 0x20 and above is for user defined interrupt.
  * IRQx is mapped to vector `IRQx_VECTOR`, which starts from `0x20 + 0x10`.
  * It installs vectors for all user-defined interrupts, using the array
    `interrupt`.

## Privilege

* In an IDT entry, there is a 16-bits segment selector and a 2-bits DPL.
  * The selector decides thew new segment
  * The DPL decides the lowest privilege that can interrupt
  * `cs` gives CPL.  If `CPL >= PL of new segment` and `CPL <= DPL`, interrupt
    succeeds.  The idea is so that the interrupt handler has higher privilege
    than the program caused the interrupt and the program has enough privilege
    to cause the interrupt.
  * `cs:eip` becomes `segment selector:offset` in the IDT entry.
* Kernel calls `set_system_trap_gate` to set syscall interrupt
  * The DPL is `0x3`, user mode
  * Thew segment selector is `__KERNEL_CS`, which is `GDT_ENTRY_KERNEL_CS << 3`.
    That is, the new segment will be kernel code segment in GDT with highest
    RPL and CPL.
* In `setup_gdt` in `boot/pm.c`, `BOOT_CS` and `BOOT_DS` GDT have acess flags
  `0x9b` and `0x93`.  Both indicate highest privilege.

## arm assembly

* `ldr r0, =0xhhhhhhhh`, if the constant is representable by mov, mov is used,
  otherwise, ldr is used

When an interrupt happens, cli (disable interrupt) automatically by CPU.  Then
* do_IRQ is called, which...
* irq_enter (increase preempt_count by HARDIRQ_OFFSET)
* irq_to_desc to get struct irq_desc *, and then desc->handle_irq
* irq_exit is called
* iret sets FLAGS

In start_kernel,
* init_IRQ is called to setup irq_desc, irq_desc->handle_irq is hooked to handle_level_irq, which...
* mask_ack irq, and handle_IRQ_event
* in handle_IRQ_event, sti, call actions, and then cli
* umask

Preempt count:

By default
* PREEMPT_MASK: 0x000000ff
* SOFTIRQ_MASK: 0x0000ff00
* HARDIRQ_MASK: 0x0fff0000
* #define in_irq()		(hardirq_count())
* #define in_softirq()		(softirq_count())
* #define in_interrupt()	(irq_count()) /* hardirq count + softirq count */

* local_irq_{disable,enable,save,restore} calls raw_local_irq_xxx, which calls native_irq_xxx, which sti/cli

* preempt_count() is per-thread
* preempt_disable() increases preempt_count
* preempt_enable() decreases preempt_count; in case TIF_NEED_RESCHED is set, call preempt_schedule
* preempt_schedule inc/dec preempt_count by PREEMPT_ACTIVE before/after schedule()

* spin_lock() -> _spin_lock(); on UP, call preempt_disable(); on SMP, it then does the real stuff

irq on s3cxxxx
* interrupt -> set SRCPND -> arbitration -> set INTPND or FIQ or nothing
* INTMSK masks interrupts
* my guess: arbitration happens when INTPND or SRCPND change.
* when an (non-fiq) interrupt happens, SRCPND is set and arbitration happens, which sets INTPND
* If INTPND is cleared before SRCPND, INTPND is set again!
* So one should clear SRCPND and then INTPND
