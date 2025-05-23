Kernel and IRQ
==============

## Initialization

- modern archs define `CONFIG_SPARSE_IRQ`
  - `NR_IRQS` is not meaningful
- `early_irq_init` allocates some `struct irq_desc`
  - this is arch-specific.  On arm64, none.
- `init_IRQ` is also arch-specific
   - it is expected to allocate per-cpu interrupt stacks and enables the irq
     controller
   - some archs use `irqchip_init`
     - it scans OF and ACPI tables and matches devices
     - drivers should use `IRQCHIP_DECLARE` or `IRQCHIP_ACPI_DECLARE`
- `struct irq_domain`
  - an irq controller driver should create one or more irq domains with
    `__irq_domain_add` or other helpers
    - irq controllers and domains can be hierarchical
  - the job of a `irq_domain` is to map virtual irq numbers to hw irq numbers
    - modern irq controllers might have no linear mapping (`size` and
      `direct_max` are 0) and a huge number of hw irq numbers (huge
      `hwirq_max`)
  - `__irq_domain_alloc_irqs` allocates `struct irq_desc`
    - normally, `irq_base` is -1
    - `irq_domain_alloc_descs` finds an unused virtual irq number range,
      allocates the `irq_desc` structs, and sets up `irq_to_desc` mappings
    - `irq_domain_alloc_irqs_hierarchy` is then called to allocate hw
      resources
      - this calls `irq_domain_ops::alloc`
      - `irq_domain_set_info` can be used to initialize `irq_desc`
    - `irq_domain_insert_irq` is called on each virq to set up hwirq-to-virq
      mapping

## HW Interrupt

- some archs have a root irq handler that is set with `set_handle_irq`
  - for example, when `gic_of_init` is called on arm64, it adds the irq domain
    and sets the root irq handler to `gic_handle_irq`
- on interrupt exception, the arch-specific exception handler is called
  - it gets the hwirq number
    - on x86, `common_interrupt` is called with the hwirq
    - on arm, `gic_handle_irq` gets hwirq from the controller
  - it maps hwirq to `irq_desc`
    - there are generic helpers such as `generic_handle_domain_irq`
    - on x86, `common_interrupt` gets the `irq_desc` from the per-cpu
      `vector_irq` array
  - `handle_irq_desc` or `generic_handle_irq_desc` calls `desc->handle_irq`,
    which is typically one of
    - `handle_level_irq`
    - `handle_fasteoi_irq`
    - `handle_edge_irq`
    - `handle_edge_eoi_irq`
    - `handle_simple_irq`
    - more
  - taking `handle_level_irq` as an example
    - `irq_mask` masks the irq, such that the line does not trigger exception
    - `irq_ack` acks the irq, such that the line is no longer asserted
      - the line can be asserted again at any time
    - `handle_irq_event`
      - `__handle_irq_event_percpu` calls `action->handler` for all actions
        - if a `action->handler` returns `IRQ_WAKE_THREAD`,
          `__irq_wake_thread` wakes up its kthread
    - `irq_unmask` unmasks the irq, such that the line triggers exception
- if an action has a `thread_fn`, it spawns a kthread running `irq_thread`
  - it sleeps until woken up by `__irq_wake_thread`
  - `irq_thread_fn` invokes `action->thread_fn`

## `request_irq`

- all variants call `request_threaded_irq`
  - `request_irq`
  - `devm_request_irq`
  - `devm_request_threaded_irq`
- `request_threaded_irq`
  - `irq_to_desc` looks up `irq_desc` from virq
  - an `irqaction` is allocated to save `handler`, `thread_fn`, `irqflags`,
    `devname`, and `dev_id`
  - `irq_chip_pm_get` enables pm
  - `__setup_irq` sets up irq
- `__setup_irq`
  - `setup_irq_thread` creates a kthread `irq/%d-%s`, if there is `thread_fn`
    - the kthread executes `irq_thread`
    - it calls `sched_set_fifo` to use `SCHED_FIFO` with `sched_priority` of
      50
  - if `IRQF_SHARED` is set, `desc->action` can have a chain of actions
  - if this is the first action of `desc->action`,
    - `irq_activate` calls `irq_domain_activate_irq` to activate the line
      recursively from the root
      - on x86, `x86_vector_activate` sets `vector_irq[hwirq]` to the `irq_desc`
  - the `irqaction` is added to `irq_desc`
  - `irq_pm_install_action` updates pm state
  - `register_irq_proc` creates `/proc/irq/<irq>`
  - `register_handler_proc` creates `/proc/irq/<irq>/<action-name>`

## Preempt count

- `preempt_count` is arch-specific
  - on x86, it returns `pcpu_hot.preempt_count`
  - on arm64, it returns `current_thread_info()->preempt.count`
- By default
  - `PREEMPT_MASK:             0x000000ff`
  - `SOFTIRQ_MASK:             0x0000ff00`
  - `HARDIRQ_MASK:             0x000f0000`
  - `NMI_MASK:                 0x00100000`
  - `PREEMPT_NEED_RESCHED:     0x80000000`
  - `foo_count()` returns `preempt_count() & foo_MASK`
    - where `foo` is `nmi`, `hardirq`, `softirq`, or `irq` to mean any of them
- on interrupt exception, the exception handler should call `irq_enter` and
  `irq_exit`
  - they increase/decrease the hardirq count
- `preempt_disable()` and `preempt_enable()` increments/decrements `preempt_count`
  - when `preempt_enable` decrements to 0, `__preempt_schedule` is called
    - it might schedule away the current task
- preemption
  - userspace has always been preemptible
    - the kernel can interrupt a task running in userspace anytime
  - kernel preemption
    - the kernel can interrupt a task running in kernel space anytime
    - unless `preempt_disable` temporarily disables preemption
  - `CONFIG_PREEMPT_NONE`
    - the kernel space is not preemptible
    - `might_sleep` is nop
  - `CONFIG_PREEMPT_VOLUNTARY`
    - the kernel space is not preemptible
    - `might_sleep` can voluntarily re-schedule
  - `CONFIG_PREEMPT`
    - implies `CONFIG_PREEMPTION` and `CONFIG_PREEMPT_COUNT`
    - the kernel space is preemptible
    - `might_sleep` is nop
  - `CONFIG_PREEMPT_RT`
    - implies `CONFIG_PREEMPTION` and `CONFIG_PREEMPT_COUNT`
    - the kernel is real-time
    - `might_sleep` is nop
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
