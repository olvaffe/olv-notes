# Scheduler

## Overview

- each task consists of a sequence of instructions to execute
- a task is runnable if nothing prevents it from running
- the CPU scheduler assigns runnable tasks to CPUs
- a CPU can have multiple runnable tasks, time-sharing the CPU
- when a CPU has no runnable task, it runs the "idle" task.  The CPU is
  considered idle

## Context Switch

- a running task calls `schedule` to context-switch voluntarily
- a running task can also be preempted involuntarily
  - an event calls `resched_curr` (or `resched_curr_lazy`) to set
    `TIF_NEED_RESCHED` for the running task
    - when hz ticks, `sched_tick` may set the flag
    - when a task is added to a rq (runqueue), `wakeup_preempt` may set the flag
      (e.g., when the new task has a higher priority)
  - when the cpu reaches preemption point, it checks for `TIF_NEED_RESCHED`
    and calls `__schedule(SM_PREEMPT)`
    - upon irq exit, `raw_irqentry_exit_cond_resched` calls `preempt_schedule_irq`
    - upon preemption re-enable, `preempt_enable` calls `preempt_schedule`
- `__schedule`
  - `pick_next_task` picks the next task to switch to
    - `pick_task` tries each sched class, from high prio to low
      - `stop_sched_class` is for `cpu_stop_threads`, for cpu hotplug
      - `dl_sched_class` is for deadline tasks
      - `rt_sched_class` is for RT tasks
      - `fair_sched_class` is for most tasks unless SCX is enabled
      - `ext_sched_class` is for `SCHED_CLASS_EXT` (SCX)
      - `idle_sched_class` is for idle tasks
  - `context_switch` switches context
    - when it returns, the cpu has switched to the next task
    - `switch_to` is arch-specific
      - it saves regs of the old task
      - it restores regs of the new task
      - it returns to the new task

## Execution Contexts

- a cpu can be in different execution contexts
  - `in_nmi` is in nmi context
    - upon hw nmi, `__nmi_enter`/`__nmi_exit` adds/subs `NMI_OFFSET`
      - they also update `HARDIRQ_OFFSET` and thus imply hardirq context
  - `in_hardirq` is in hardirq context
    - upon hw irq, `__irq_enter_raw`/`__irq_exit_raw` adds/subs `HARDIRQ_OFFSET`
      - sometimes `HARDIRQ_OFFSET` is added/substracted directly
  - `in_serving_softirq` is in softirq context
    - inside `handle_softirqs`, `softirq_handle_begin`/`softirq_handle_end`
      adds/subs `SOFTIRQ_OFFSET`
    - note that `handle_softirq` enables local irq which is the key difference
      - hardirq handlers are called with local irq disabled (unless nesting)
      - toward the end of hardirq, `handle_softirq` calls softirq handlers
        with local irq enabled
  - `in_task` is in task context
    - this returns true when the rest returns false
- `current` is per-cpu and is always non-NULL
  - it points to the running task when `in_task`
  - it points to the preemptied task in other contexts
- a cpu is in an atomic context when it must not sleep
  - this happens when scheduling is not possible or utterly wrong (e.g.,
    deadlock), because sleeping implies scheduling
  - `in_atomic` does not detect all atomic contexts
    - with `CONFIG_DEBUG_ATOMIC_SLEEP`, `__might_sleep` tries to detect more
  - `in_nmi`, `in_hardirq`, and `in_serving_softirq` are atomic contexts
  - `in_task` can be an atomic context when within
    - `local_irq_disable` and `local_irq_enable`
    - `local_bh_disable` and `local_bh_enable`
    - `preempt_disable` and `preempt_enable`
    - `spin_lock` and `spin_unlock` (because they imply preempt disable)
    - more

## Idle Tasks

- most of the time, each cpu core executes the idle thread `swapper/<cpuid>`
  (`INIT_TASK_COMM`) and is inside `cpu_startup_entry`
  - boot cpu in `start_kernel`
    - `sched_init` calls `init_idle` to init `init_task` (pid 0)
      - `current` is `init_task` statically-defined in `init/init_task.c`
    - `rest_init` forks pid 1 to run `kernel_init`
    - `cpu_startup_entry` enters the idle loop
  - boot cpu in `kernel_init`
    - `smp_init` forks idle tasks and boot non-boot cpus
      - on x86, `common_cpu_up` points `current_task` to idle task for
        non-boot cpus
  - non-boot cpu in `start_secondary`
    - `cpu_startup_entry` enters the idle loop
- `cpu_startup_entry` calls `do_idle` in an infinite loop
  - when there is nothing to do, `cpuidle_idle_call` enters lower and lower
    idle states
    - this calls into the hw-specific cpuidle driver
  - when an idle CPU is assigned a task by another CPU, the other CPU calls
    `ttwu_queue` to queue the task to the idle CPU
    - when the two CPUs do not share LLC (i.e., in different packages), the
      idle CPU will mark the task `TASK_RUNNING` in `sched_ttwu_pending`
      - unless the idle CPU polls the need-resched flag, an IPI is sent to wake
       up the idle CPU, which will call `scheduler_ipi`
    - otherwise, the calling CPU marks the task `TASK_RUNNING` directly for
      the idle CPU.  Unless the idle CPU polls the need-resched flag, an IPI
      is also sent to wake it up.
  - the idle CPU exits idle state and returns from `cpuidle_idle_call`
    - because `need_resched` is true, it calls `schedule_idle` to
       context-switch to the task
  - the CPU context-switches back to `swapper/<cpuid>`
    - when the running task calls `schedule`, and there is no runnable task,
      `pick_task_idle` picks the idle task and context-switches to it
    - the cpu runs `do_idle` loop of the idle task again
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

## Preemption Configs

- on interesting archs,
  - `CONFIG_ARCH_HAS_PREEMPT_LAZY` is set
  - `CONFIG_ARCH_NO_PREEMPT` is cleared
- `kernel/Kconfig.preempt`
  - `CONFIG_PREEMPT_NONE` is unavailable
  - `CONFIG_PREEMPT_VOLUNTARY` is unavailable
  - `CONFIG_PREEMPT` aka full or low-latency preemption
  - `CONFIG_PREEMPT_LAZY` aka lazy preemption
- when a task is in user mode, it can be preempted when
  - it runs out of time
  - a higher priority task is scheduled
  - on return-to-user path, `exit_to_user_mode_loop` checks for
    `TIF_NEED_RESCHED`/`TIF_NEED_RESCHED_LAZY` and calls `schedule`
- when a task is in kernel mode,
  - `CONFIG_PREEMPT_VOLUNTARY`: it is never preemptied
    - `preempt_disable` and `preempt_enable` are essentially nops
    - `preempt_count` remains `INIT_PREEMPT_COUNT` (1) forever
    - instead, when the task calls `might_sleep` or `cond_resched`, it calls
      down to `preempt_schedule_common` to schedule voluntarily
  - `CONFIG_PREEMPT`: it gets preempted similar to when in user mode,
    unless when preemption is disabled (e.g., in critical sections)
    - there are in-kernel preemption points such as
      `raw_irqentry_exit_cond_resched` or `preempt_enable` that checks for
      `TIF_NEED_RESCHED` and calls `__schedule(SM_PREEMPT)`
  - `CONFIG_PREEMPT_LAZY`: it is similar to `CONFIG_PREEMPT`, except
    - CFS calls `resched_curr_lazy` to set `TIF_NEED_RESCHED_LAZY`, instead of
      `resched_curr` to set `TIF_NEED_RESCHED`, in a few cases
      - this waits until `sched_tick` to upgrade `TIF_NEED_RESCHED_LAZY` to
        `TIF_NEED_RESCHED`
    - other events still call `resched_curr` to set `TIF_NEED_RESCHED`
      immediately

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

## Run Queue

- system task tracking
  - there is statically-initialized `init_task`
  - all other tasks are duplicated by `dup_task_struct`
  - all processes are tracked in `init_task.tasks`
    - `for_each_process` loops through all processes
    - `for_each_process_thread` loops through all threads of all tasks
    - this excludes `init_task` and idle threads
  - for `/proc`, `proc_pid_readdir` calls `next_tgid` to loop through all pids
    - because `init_task` and idle threads do not allocate pids from the
      namespace, they are excluded
- scheduler task tracking
  - scheduler uses rqs to tracks task
    - a task not on any rq is not visible to scheduler
  - `init_idle` is called on `init_task` and idle threads
    - `rq->curr` is set to `idle`
    - `idle->on_rq` is set to `TASK_ON_RQ_QUEUED`
  - `wake_up_new_task` is called on other newly created tasks
- `__schedule` can remove a task from rq
  - `prev` is `rq->curr`, the current running task
  - if `!preempt` and `prev_state`, `try_to_block_task` tries to remove the
    task from rq
    - the condition means
      - `__schedule` is called voluntarily, not by preemption
      - the task state is not `TASK_RUNNING` (0)
    - if the condition is false, `try_to_block_task` is not called and this
      works like `yield`
    - `dequeue_task` calls `dequeue_task_fair`
      - `dequeue_entity` calls `__dequeue_entity` to erase the task from the
        rb tree
    - `__block_task` clears `p->on_rq`
    - once removed, the task is never considered until `try_to_wake_up` adds
      it back to rq
  - `pick_next_task` picks the next task from rq
    - it might pick the same task if `prev` is still on rq
- `try_to_wake_up` (ttwu) can add a task to rq
  - `p->__state` is set to `TASK_WAKING`
  - `ttwu_queue` calls `ttwu_do_activate`
    - `activate_task`
      - `enqueue_task_fair` calls `__enqueue_entity` to add the task to the rb
        tree
      - `p->on_rq` is set to `TASK_ON_RQ_QUEUED`
    - `ttwu_do_wakeup` sets `p->__state` to `TASK_RUNNING`
- `__schedule` and `try_to_wake_up` are atomic against each other
  - `try_to_wake_up` and `set_current_state` have big comments
  - if `__schedule` goes before `try_to_wake_up`,
    - after `__schedule` and if `try_to_block_task` is called,
      - `p->__state != TASK_RUNNING`
      - `p->on_rq == 0`
    - after `try_to_wake_up`,
      - `p->__state == TASK_RUNNING`
      - `p->on_rq == 1`
  - if `try_to_wake_up` goes before `__schedule`,
    - after `try_to_wake_up`,
      - `p->__state == TASK_RUNNING`
      - `p->on_rq == 1`
    - `__schedule` does not call `try_to_block_task`
- voluntary sleep
  - a task goes to sleep voluntarily via
    - `while (true)`
      - `set_current_state(TASK_INTERRUPTIBLE or TASK_UNINTERRUPTIBLE);`
      - `if (condition) break;`
      - `schedule();`
    - `__set_current_state(TASK_RUNNING);`
  - another task wakes it up via
    - `condition = true`
    - `try_to_wake_up()`
  - the idea is for `schedule` to remove the task from rq and for
    `try_to_wake_up` to add it back
  - what if `schedule` and `try_to_wake_up` race, and `schedule` goes later
    - `schedule` does not call `try_to_block_task`
    - it exits the loop on next iteration

## Wait Queue

- basic usage
  - `init_waitqueue_head` inits a queue
  - when a thread needs to wait for a condition, `wait_event` on the queue
  - when another thread changes the condition, `wake_up` on the queue
- manual wait
  - when a thread needs to wait for a condition,
    - `init_waitqueue_entry` inits an entry
    - `add_wait_queue` adds the entry to the queue
    - loop
      - `set_current_state` sets to `TASK_INTERRUPTIBLE` or `TASK_UNINTERRUPTIBLE`
      - check for condition
      - `schedule` sleeps
      - check for condition
    - `__set_current_state` sets to `TASK_RUNNING`
    - `remove_wait_queue` removes the entry
- `init_waitqueue_head` inits a `wait_queue_head`
  - it consists of a spinlock and a list
- `init_waitqueue_entry` inits a `init_waitqueue_entry`
  - `flags` is 0
  - `private` is the task
  - `func` is `default_wake_function`, which calls `try_to_wake_up`
- `add_wait_queue` adds an entry to a queue
  - it locks the queue to add the entry to the list
- `remove_wait_queue` removes an entry from a queue
  - it locks the queue to delete the entry from the list
- `wake_up` wakes up a queue
  - it locks the queue
  - it calls `func` of each entry with the lock held
    - return values
      - negative: stop immediately
      - 0: not woken up
      - 1: woken up
    - if woken up, and is `WQ_FLAG_EXCLUSIVE`, stop immediately as well
      - when N thread waits for an event, and only one thread is needed to
        handle each event, they should set `WQ_FLAG_EXCLUSIVE`
  - it unlocks the queue

## `completion`

- `struct completion` consists of
  - `unsigned int done` as the done counter
  - `struct swait_queue_head wait` as the wait queue
    - the wait queue has a spinlock to protect itself and `done`
- `init_completion` inits `done` to 0 and inits the wait queue
- `complete` increments `done` and wakes up the first task
- `wait_for_completion*` waits for `done` and decrements it
  - they all call `__wait_for_common`, with these defaults
    - `action` is `schedule_timeout`
    - `timeout` is `MAX_SCHEDULE_TIMEOUT`
    - `state` is `TASK_UNINTERRUPTIBLE`
  - it calls `action` in a loop until `done > 0` or `timeout == 0`
  - if `done > 0`, it decrements `done`
  - it returns remaining time (0 means timeout)
- `try_wait_for_completion` decrements `done` without waiting
  - if `done == 0`, it early returns without decrementing
- `completion_done` checks if `done != 0`
- `complete_all` sets `done` to `UINT_MAX` and wakes up all tasks
- `reinit_completion` re-inits `done` to 0, for use after `complete_all`

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
