Kernel and IRQ
==============

## Flow

- when a device needs to be serviced, it assert an interrupt line
- upon line assertion, irq controller latches the signal and raises a cpu
  exception
  - on x86, io apic receives the signal from the device, latches the signal,
    and one of the per-core local apic forwards the signal to the cpu core
- some cpu core handles the exception
  - it enters irq context with irq disabled locally
  - it asks the controller to mask out the line, such that the line cannot
    trigger exception
  - it asks the controller to ack the line, such that the controller can latch
    another signal from another line
  - it re-enables irq locally for nested irq (optional)
  - it services the device such that the device no longer asserts the line
    - this usually involves reading and acking device reqs, but real handling
      of device reqs happen in bottom half
  - it asks the controller to unmask the line
    - the line may be asserted and raises another exception, if the device, or
      another device sharing the line, has another request
  - it exits irq context with irq enabled locally
- the cpu core performs minimal work for exception handling for several reasons
  - it wants to get back to normal operation asap
    - there may be other exceptions to handle, or processes to run
  - it wants to unmask the line asap
    - other devices may share the same line

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

## `request_irq`

- all variants call `request_threaded_irq`
  - `request_irq`
  - `devm_request_irq`
  - `devm_request_threaded_irq`
- `request_threaded_irq`
  - `irq_to_desc` looks up `irq_desc` from virq
  - an `irqaction` is allocated to save `handler`, `thread_fn`, `irqflags`,
    `devname`, and `dev_id`
    - if there is no handler, it defaults to `irq_default_primary_handler`
      which is often used with `IRQF_ONESHOT`
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
    - `irq_ack` acks the irq, such that the chip can handle other lines
    - `handle_irq_event`
      - `__handle_irq_event_percpu` calls `action->handler` for all actions
        - if a `action->handler` returns `IRQ_WAKE_THREAD`,
          `__irq_wake_thread` wakes up its kthread
    - `irq_unmask` unmasks the irq, such that the line triggers exception
- if an action has a `thread_fn`, it spawns a kthread running `irq_thread`
  - it sleeps until woken up by `__irq_wake_thread`
  - `irq_thread_fn` invokes `action->thread_fn`
- `IRQF_ONESHOT` keeps the line masked until the threaded handler returns
  - during `__setup_irq`, the bit causes
    - `new->thread_mask = 1UL << ffz(thread_mask);`
    - `desc->istate |= IRQS_ONESHOT`
  - during `handle_irq_event`, each handler is called, and if
    `IRQ_WAKE_THREAD` is returned, `__irq_wake_thread` sets
    - `desc->threads_oneshot |= action->thread_mask;`
  - during `irq_thread_fn`, the thread fn is called, and
    `irq_finalize_oneshot` sets
    - `desc->threads_oneshot &= ~action->thread_mask;`
    - `unmask_threaded_irq` unmasks the line
