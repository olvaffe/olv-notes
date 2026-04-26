# Kernel and IRQ

## Root Interrupt Controller

- modern archs define `CONFIG_SPARSE_IRQ`
  - `NR_IRQS` is not meaningful
- `early_irq_init`
  - if there are legacy irqs, it may pre-allocate some `struct irq_desc`s
    - modern way allocs `irq_desc` on demand when the device driver enables irq
  - `arch_early_irq_init`
    - on x86, this creates the root irq domain, `x86_vector_domain`
      - the chip is `lapic_controller`
- `init_IRQ`
  - on x86, `native_init_IRQ`
  - on arm, `irqchip_init` scans OF/ACPI to init matching devices
    - drivers are defined by `IRQCHIP_DECLARE` or `IRQCHIP_ACPI_DECLARE`
    - `gic_of_init` creates the root irq domain, `gic_data.domain`
      - the chip is `gic_chip`
      - it also sets the root irq handler to `gic_handle_irq`

## Dummy Controller

- it is easier to see what's going on if there is a dummy interrupt controller
  between the root interrupt controller and the device
- the driver of the dummy interrupt controller would look like
  - `dummy_probe`
    - `dummy_domain = irq_domain_create_linear(NULL, 1, &dummy_ops, NULL)`
      - `dummy_ops.map` calls `irq_set_chip_and_handler(virq, &dummy_irq_chip, handle_simple_irq)`
      - both `dummy_irq_chip` and `handle_simple_irq` are provided by irq core
    - `root_virq = irq_create_mapping(root_domain, root_hwirq)`
      - we are a client of the root controller
      - we typically call `platform_get_irq_byname` instead, which looks
        `root_domain` and `root_hwirq` up from dt
    - `request_irq(root_virq, dummy_handler, ...)`
      - `dummy_handler` looks like
        `generic_handle_domain_irq(dummy_domain, dummy_hwirq); returns IRQ_HANDLED;`
  - `dummy_remove`
    - `free_irq(root_virq, ...)`
    - `irq_dispose_mapping(dummy_virq)`
    - `irq_domain_remove(dummy_domain)`
- our client would use us like how we use the root controller
  - when they calll `irq_create_mapping(dummy_doman, dummy_hwirq)`,
    - irq core allocs a new `irq_desc` from a global pool and `virq` is the
      index to the pool
    - irq core also calls `dummy_ops.map`, which updates irq desc to
      `dummy_irq_chip` and `handle_simple_irq`
  - when they call `request_irq(dummy_virq, dev_handler, ...)`,
    - irq core updates irq desc with their irq handler
  - when they call `free_irq(dummy_virq, ...)`,
    - irq core updates irq desc to remove their irq handler
- when they assert an interrupt to us, we assert an interrupt to the root
  controller; irq core calls `dummy_handler`
  - `generic_handle_domain_irq` looks up the irq desc and calls
    `handle_simple_irq` which ultimately calls `dev_handler`
    - `dev_handler` acks the dev interrupt

## Chained Controllers

- cascaded interrupt controllers use so called chained handlers
  - they do not use hierarchical irq domains
- `irq_set_chained_handler` is called on the parent virq
  - when the parent virq fires, the handler finds the child domain and reads
    the child hwirq to call `generic_handle_domain_irq`

## MSI and Hierarchical IRQ Domains

- MSI is always hierarchical
  - the msi controller provides the msi parent domain
    - x86 calls `x86_create_pci_msi_domain` to create the msi parent domain
      - `native_create_pci_msi_domain` returns `x86_vector_domain`
        - that is, the msi parent domain is the same as the system root domain,
          which is the local apic domain
      - when a pci device is enumerated, `pcibios_device_add` sets the msi parent
        domain to `x86_pci_msi_default_domain` which is `x86_vector_domain`
    - arm calls `msi_create_parent_irq_domain` to create the msi parent domain
      - `gic_of_init -> gic_init_bases -> its_init -> ... -> its_init_domain`
        - the msi parent domain is a child of the system root domain
      - when a pci device is enumerated, `pci_set_msi_domain` sets the msi parent
        domain
  - the msi devices provide the msi device domains
    - when a pci driver probes a pci device, it calls
      `msi_create_device_irq_domain` to create the msi device domain
      - `pci_alloc_irq_vectors`, `pci_enable_msi`, etc. all call down to
        `pci_create_device_domain` which calls `msi_create_device_irq_domain`
        - the template is `pci_msix_template` or `pci_msi_template`
- `msi_create_device_irq_domain`
  - when a pci driver enables msi, this is called from
    `pci_create_device_domain`
  - the caller provides a template, such as `pci_msix_template`
  - `pops->init_dev_msi_info`
    - on x86, `x86_init_dev_msi_info`
      - it sets `info->handler` to `handle_edge_irq`
    - on arm, `its_init_dev_msi_info`
- `msi_domain_alloc_irqs_all_locked`
  - when a pci driver enables msi, this is called from
    `pci_msi_setup_msi_irqs`
  - `__msi_domain_alloc_irqs` calls `__irq_domain_alloc_irqs`
    - `irq_domain_alloc_descs` allocs the `irq_desc`
    - `irq_domain_alloc_irq_data` allocs the `irq_data` chain
    - `irq_domain_alloc_irqs_hierarchy` calls `msi_domain_alloc`
      - `irq_domain_alloc_irqs_parent` allocs from the parent domain
        - on x86, `x86_vector_alloc_irqs`
          - it inits the parent irq chip to `lapic_controller`
        - on arm, `its_irq_domain_alloc`
          - `its_irq_gic_domain_alloc` calls into the root domain
            - `gic_irq_domain_alloc` inits the root irq chip to `gic_chip` and
              the irq desc handler to `handle_fasteoi_irq`
          - it inits the parent irq chip to `its_irq_chip`
      - `msi_domain_ops_init`
        - it sets the irq data chip to `info->chip`, which is from
          `pci_msix_template`, etc.
        - it sets irq desc handler to `info->handler`, which is
          `handle_edge_irq` on x86

## IRQ Domain

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

## HW Flow

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

## HW Interrupt

- some archs have a configurable root irq handler that is set with `set_handle_irq`
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

## `irq_work`

- `irq_work` is a bottom-half mechanism that executes in hardirq ctx
- but why moves the work from hardirq ctx to another hardirq ctx?
  - NMI: it moves the work from unsafe nmi hardirq ctx to standard hardirq ctx
  - deadlock: when `printk` prints to a console that has a nested `printk`,
    defer the nested `printk`
- `init_irq_work` inits an irq work
- `irq_work_queue` queues an irq work (from hardirq ctx)
  - it adds the work to the global `raised_list`
  - `irq_work_raise` raises a arch-specific ipi
- x86 `arch_irq_work_raise` raises `IRQ_WORK_VECTOR` ipi to itself
  - when the local irq is enabled, `sysvec_irq_work` handles the ipi and calls
    `irq_work_run`
- arm `arch_irq_work_raise` raises `IPI_IRQ_WORK` ipi to itself
  - when the local irq is enabled, `do_handle_IPI` handles the ipi and calls
    `irq_work_run`
- `irq_work_run` calls `irq_work_single` on each work on `raised_list`
