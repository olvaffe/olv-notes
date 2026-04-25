# Signal

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

- `__exit_to_user_mode_prepare` is called on various exit paths, such as
  `syscall_exit_to_user_mode` or `irqentry_exit_to_user_mode`, to handle
  signals
  - `exit_to_user_mode_loop` handles various works
  - if `_TIF_SIGPENDING`, `arch_do_signal_or_restart` handles signals
    - it typically calls `get_signal` followed `handle_signal`
- `get_signal` returns the next pending signal for handling
  - `dequeue_synchronous_signal` or `dequeue_signal`
    - it removes a `sigqueue` from `pending->list`
    - if this is the last instance, it clears the bit from `pending->signal`
    - `recalc_sigpending` clears `_TIF_SIGPENDING` if no more pending signal
  - it looks up the action in `sighand`
    - `SIG_IGN` ignores the signal and loop to dequeue the next
    - `SIG_DFL` is the in-kernel default handling
    - otherwise, it returns the dequeued signal to the caller
- arch-specific `handle_signal` calls `setup_rt_frame` and `signal_setup_done`
  - `setup_rt_frame` sets up a frame in the user stack, such that when we
    return to userspace, we will return to the userspace signal handler; And
    when the signal handler returns, it will jump back to kernel
    - on arm, `setup_return`
      - `regs->pc` is set to `ksig->ka.sa.sa_handler`
      - `regs->sp` is set to `user->sigframe`
      - `regs->regs[30]` (link reg) is set to `sigtramp` in VDSO
        - `__kernel_rt_sigreturn` makes `rt_sigreturn` syscall
    - on x86, `x64_setup_rt_frame`
      - `regs->ip` is set to `ksig->ka.sa.sa_handler`
      - `regs->sp` is set to `frame`
      - the return addr is set to `ksig->ka.sa.sa_restorer`
        - unlike arm, it is provided by userspace libc
        - its job is to make `rt_sigreturn` syscall
  - `signal_setup_done` calls `signal_delivered`
- with the setup, we exit to userspace signal handler directly
- when the userspace signal handler returns, it returns to a userspace
  trampoline that calls `rt_sigreturn`
- `SYSCALL_DEFINE0(rt_sigreturn)` is called after userspace signal handler
  - on x86, `restore_sigcontext` restores `regs` from `frame->uc.uc_mcontext`
  - on arm, `restore_sigframe` restores `regs` from `sf->uc.uc_mcontext`
  - when the syscall returns, it returns to where the userspace was before
    the signal

## Syscall Interruption and Restart

- when a syscall sleeps inside `wait_event_interruptible`, and another process
  sends a signal,
  - the other process calls `complete_signal` to wake up the sleeping process
  - `prepare_to_wait_event` detects the signal and propagates `-ERESTARTSYS`
  - the syscall fails with `-ERESTARTSYS`
- on the exit path, `syscall_exit_to_user_mode` calls
  `arch_do_signal_or_restart` to handle the signal
  - if `SA_RESTART` is set for the signal,
    - arm decrements `reg->pc` by 4 such that userspace re-executes `SVC`
      instr after signal handling and returning to userspace
    - x86 decrements `reg->ip` by 2 such that userspace re-executes `SYSCALL`
      instr similarly
  - if `SA_RESTART` is cleared for the signal,
    - arm sets `regs->regs[0]` to `-EINTR` as the return value of the syscall
    - x86 sets `reg->ax` to `-EINTR` similarly
