Signal
======

## `kill`

- `SYSCALL_DEFINE2(kill, ...)`
- `prepare_kill_siginfo` wraps the signal in a `kernel_siginfo`
- `group_send_sig_info` sends the signal to a task
  - `prepare_signal` checks if the signal is blocked or has side effects
  - `pending` is set to per-thread `t->pending` or per-process
    `t->signal->shared_pending`
  - `legacy_queue` checks if the signal is already pending
  - `sigqueue_alloc` allocs a `sigqueue`
  - it adds the `sigqueue` to `pending->list`
  - `signalfd_notify` notifies signalfd if any
  - `sigaddset` sets the signal bit in `pending->signal`
  - `complete_signal`
    - it picks a task to handle the signal
    - if fatal, injects `SIGKILL` to all threads and wakes them up
    - else, `signal_wake_up` wakes up the target task
      - it sets `TIF_SIGPENDING`
      - `wake_up_state` calls `try_to_wake_up` to add it to rq
        - if the task is already running, `kick_process` sends an ipi to force
          a timely signal handling

## Signal Handling

- if the task is really woken up from sleep,  it will notice the pending
  signals and process them first after being scheduled
- if the task is running, it will be force-rescheduled

## `EINTR`

- `SYSCALL_DEFINE2(nanosleep, ...)` puts the current task into
  `TASK_INTERRUPTIBLE` state in `do_nanosleep`
  - it starts the timer and schedules the current task away
  - but if a signal comes in before the timer expires, the task is woken up
    and is back to `TASK_RUNNING`
  - the caller of the syscall gets an `EINTR`

## Miscs

- eventfd
- signalfd
- timerfd
- memfd
- userfaultfd
- pidfd
