Kernel and softirq
==================

## softirq

- softirq is not for drivers
  - it is for a few special subsystems
  - unlike hardirq, where local irq is disabled, `__do_softirq` handles
    softirqs with local irq enabled
- `spawn_ksoftirqd` spawns a `ksoftirqd/%u` thread for each cpu
- `raise_softirq` raises a softirq
  - it calls `or_softirq_pending` to set the pending bit
  - if called from the interrupt context, that's all
  - otherwise, it also wakes up ksoftirqd to handle the softirq
    - ksoftirqd can run on another cpu
- `invoke_softirq` handles softirqs or wakes up ksoftirqd
  - `force_irqthreads` is normally false
  - it calls `__do_softirq` to handle softirqs unless ksoftirqd is already
    doing the work
  - it is called from a few places such as `irq_exit`
- `__do_softirq`
  - `softirq_handle_begin` increments `softirq_count`
  - actions are called with local irq enabled
  - `softirq_handle_end` decrements `softirq_count`
