Kernel and IRQ
==============

## IRQ Domain

- each irq controller is a `irq_domain`
  - irq controllers and domains are hierarchical
- kernel uses virtual irq numbers
  - there are `NR_VECTOR` hardware vectors
  - however, `NR_IRQS` can be thousands or tens of thousands
- for an irq controller with N hwirqs, `__irq_domain_alloc_irqs` is called
  - normally, `irq_base` is -1
  - `irq_domain_alloc_descs` finds an unused virtual irq number range,
    allocates the `irq_desc` structs, and sets up `irq_to_desc` mappings
  - `irq_domain_alloc_irqs_hierarchy` is then called to allocate hw resources
    - this initializes `irq_desc::irq_data`, especially `hwirq` and `chip`
      fields recursively
  - `irq_domain_insert_irq` is called on each virq to set up hwirq-to-virq
    mapping
- `request_irq` requests a virq
  - `irq_to_desc` to look up the `irq_desc`
  - `irq_domain_activate_irq` is called to activate the line recursively from
    the root
  - on x86, this sets `vector_irq[hwirq]` to the `irq_desc`
  - `do_IRQ` handles normal device IRQs on x86.  It gets the `irq_desc` from
    the per-cpu `vector_irq` array and call `irq_desc::handle_irq`.  The
    handler is usually one of `handle_edge_irq` or `handle_level_irq`
  - the handler specified in `request_irq` is internally known as an
    `irqaction`.  `handle_irq_event` invokes all actions.

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
