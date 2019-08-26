Scheduler
=========

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

## Wait Queue

- `wait_event_interruptible` marks self `TASK_INTERRUPTIBLE` and keeps calling
  `schedule` until the condition is true.  Then it sets itself back to
  `TASK_RUNNING`.  `init_wait_entry` adds a `wait_queue_entry` with
  `autoremove_wake_function` as the callback, which simply calls
  `try_to_wake_up`
- `wake_up_interruptible` calls the callback function, which calls
  `try_to_wake_up`
