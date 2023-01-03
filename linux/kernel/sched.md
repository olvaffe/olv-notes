Scheduler
=========

## Boot

- per-cpu `current_task` is statically initialized to statically initialized
  `init_task` defined in `init/init_task.c`
  - `comm` is `INIT_TASK_COMM` (swapper)
  - `active_mm` is `init_mm`
  - `thread_pid` is `init_struct_pid` w/ pid 0
- when `start_kernel` reaches `rest_init`, it spawns two threads
  - `kernel_init` first, which has pid 1
    - it will `execve("/sbin/init")` later
  - `kthreadd` second, which has pid 2
    - kthreadd will respond to future `kthread_create` requests
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

## Wait Queue

- `wait_event_interruptible` marks self `TASK_INTERRUPTIBLE` and keeps calling
  `schedule` until the condition is true.  Then it sets itself back to
  `TASK_RUNNING`.  `init_wait_entry` adds a `wait_queue_entry` with
  `autoremove_wake_function` as the callback, which simply calls
  `try_to_wake_up`
- `wake_up_interruptible` calls the callback function, which calls
  `try_to_wake_up`

## Context Switches

- interrupt/signal/syscall/etc
- on x86, `switch_to` calls `__switch_to_asm`
  - push callee-preserved registers to stack (of prev)
  - switch stack
  - pop callee-preserved registers from stack (of next)

