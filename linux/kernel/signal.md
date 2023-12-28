Signal
======

## Signals

- signal(7)
- kill(2) sends a signal to a pid
- `kernel/signal.c`
  - adds a signal record (`sigqueue`) to the target's pending list
    (`sigpending`)
  - `signal_wake_up` to inform the target

## `kill`

- `SYSCALL_DEFINE2(kill, ...)` wraps the signal in a `kernel_siginfo` and
  eventually calls `__send_signal`
  - it adds the siginfo to the `sigpending` list of the target task
  - notifies the target's signalfd, if any, with `signalfd_notify`
  - wakes up the target task with `signal_wake_up`
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
