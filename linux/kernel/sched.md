Scheduler
=========

## Boot

- `kernel_init`
  - after `workqueue_init`, workqueue queues start running which spawn a
    kthread each
    - `rcu_gp`, `rcu_par_gp`, etc.
  - `init_mm_internals` starts `mm_percpu_wq`
  - after `do_pre_smp_initcalls`, `early_initcall`s are executed.
    - `rcu_spawn_core_kthreads` registers `rcu_cpu_thread_spec`
    - `spawn_ksoftirqd` registers `softirq_threads`
    - `cpu_stop_init` registers `cpu_stop_threads`
    - `idle_inject_init` registers `idle_inject_threads`
    - `rcu_spawn_gp_kthread` starts `rcu_gp_kthread` (comm is `rcu_preempt`)

## cpuidle

- each task consists of a sequence of instructions to execute
- a task is runnable if nothing prevents it from running
- the CPU scheduler assigns runnable tasks to CPUs
- a CPU can have multiple runnable tasks, time-sharing the CPU
- when a CPU has no runnable task, it runs the "idle" task.  The CPU is
  considered idle
- the idle task executes an idle loop.  In each iteration
  - it asks the governor in cpuidle subsystem which idle state it should enter
  - it then asks the driver in cpuidle subsystem to enter the idle state
  - the CPU does not run when it is in the idle state
  - when the CPU starts running again, it exits the idle state

## Init Idle Task (swapper)

- most of the time, a CPU executes the idle thread `swapper/<cpuid>`
  (`INIT_TASK_COMM`) and is inside `cpu_startup_entry`
  - the CPU enters idle state inside `cpuidle_idle_call`
  - when an idle CPU is assigned a task by another CPU, the other CPU calls
    `ttwu_queue` to queue the task to the idle CPU
    - when the two CPUs do not share LLC (i.e., in different packages), the
      idle CPU will mark the task `TASK_RUNNING` in `sched_ttwu_pending`
      - unless the idle CPU polls the need-resched flag, an IPI is sent to wake
      	up the idle CPU, which will call `scheduler_ipi`
    - otherwise, the calling CPU marks the task `TASK_RUNNING` directly for
      the idle CPU.  Unless the idle CPU polls the need-resched flag, an IPI
      is also sent to wake it up.
  - the idle CPU exits idle state, notices that `need_resched` is true, and
    calls `schedule_idle` to context-switch to the task
  - the CPU context-switches back to `swapper/<cpuid>`, notices that
    `need_resched` is false, and enters idle state again thanks to the
    idle loop in `cpu_startup_entry`
- in ftrace, we normally see these events in order
  - a task is woken by another CPU
    - `sched_waking: comm=Xorg pid=2498 prio=120 target_cpu=005`
  - the task wakes up and is added to the run queue and becomes runnable
    (`TASK_RUNNING`)
    - `reschedule_entry: vector=253`
    - `sched_wakeup: comm=Xorg pid=2498 prio=120 target_cpu=005`
    - `reschedule_exit: vector=253`
  - the designated CPU exits idle state (`PWR_EVENT_EXIT` is -1)
    - `cpu_idle: state=4294967295 cpu_id=5`
  - the CPU context switches to the task
    - `sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=Xorg next_pid=2498 next_prio=120`
  - the task runs for a while
  - the task goes to sleep again
    - `sched_switch: prev_comm=Xorg prev_pid=2498 prev_prio=120 prev_state=S ==> next_comm=swapper/5 next_pid=0 next_prio=120`
  - the CPU enters idle state
    - `cpu_idle: state=1 cpu_id=5`

## Preemption

- when a task is in user mode, it can be preempted when
  - it runs out of time
  - a higher priority task is scheduled
- when a task is in kernel mode,
  - `CONFIG_PREEMPT_NONE`: it gets preempted only when it needs to wait for an
    external event (e.g., I/O)
  - `CONFIG_PREEMPT_VOLUNTARY`: it gets preempted only when it might need to
    wait for an external event (where `might_sleep` is called)
  - `CONFIG_PREEMPT_LL`: it gets preempted similar to when in user mode,
    unless when preemption is disabled (e.g., in critical sections)
- one of the variants of `schedule` is called to potentially context switch to
  another task

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

## Wait Queue

- `wait_event_interruptible` marks self `TASK_INTERRUPTIBLE` and keeps calling
  `schedule` until the condition is true.  Then it sets itself back to
  `TASK_RUNNING`.  `init_wait_entry` adds a `wait_queue_entry` with
  `autoremove_wake_function` as the callback, which simply calls
  `try_to_wake_up`
- `wake_up_interruptible` calls the callback function, which calls
  `try_to_wake_up`

## man 7 sched

- conceptually,
  - the scheduler maintains a wait list of runnable tasks for each possible
    `sched_priority`
  - to determine which task to run next, it always picks the first task of the
    first non-empty wait list that has the highest `sched_priority`
- `sched_setscheduler` changes the policy and `sched_priority` of the
  specified task
  - real-time policies always have non-zero `sched_priority`
    - use `sched_get_priority_max` and `sched_get_priority_min` to find the
      valid range
  - normal policies always have zero `sched_priority`
  - the policy determines how a task is inserted or moved within its wait list
- scheduling is preemptive
  - if a task of higher `sched_priority` is ready to run, the current task is
    preemptied and returned to the wait list it belongs to
- `SCHED_FIFO` is a real-time policy
  - if a task of this policy is preemptied (by another task that has higher
    `sched_priority`), it is returned to the head of its wait list
  - if a task of this policy becomes runnable, it is added to the end of its
    wait list
  - if the task calls `sched_yield`, it is moved to the end of its wait list
  - a task of this policy runs until one of these conditions
    - it is blocked by io
    - it is preemptied by a task of higher `sched_priority`
    - it calls `sched_yield`
- `SCHED_RR` is a real-time policy
  - a task of this policy is the same as a task of `SCHED_FIFO`, with one
    exception/improvement: after running for a fixed time, it is moved to the
    end of its wait list
  - `sched_rr_get_interval` returns the fixed time limit
    - `cat /proc/sys/kernel/sched_rr_timeslice_ms` returns `90`
- `SCHED_OTHER` is a normal policy
  - it is called `SCHED_NORMAL` in the kernel
  - the task to run is based on the dynamic priority
    - the dynamic priority of a task is calculated from the nice value and is
      increased each time the task is ready to run but is not picked
  - `nice -n VAL cmd` calls `setpriority(PRIO_PROCESS, 0, VAL)` and
    `execve(cmd)`
  - `setpriority` sets the priority of the specified entity
    - `-20` is the highest
    - `19` is the lowest
    - `0` is the default
    - negative numbers require root
- `SCHED_BATCH` is a normal policy
  - it is the same as `SCHED_OTHER`, except it is assumed to be cpu-intensive
    and non-interactive
  - I guess it is less likely to be scheduled and has a longer time slice once
    scheduled
- `SCHED_IDLE` is a normal policy
  - it does not use the nice value
  - lower priority than nice 19
- RT throttling
  - a task of a real-time policy can occupy a cpu forever if it does
    `while (true);`
  - `/proc/sys/kernel/sched_rt_period_us` is the length of a period
    - it is 1 second by default
  - `/proc/sys/kernel/sched_rt_runtime_us` is the length of cpu time all RT
    tasks can use
    - it is 0.95 second by default, meaning 5% of cpu time is reserved for
      normal tasks
