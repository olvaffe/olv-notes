Kernel exit
===========

## `exit_group` syscall

- userspace `_exit` calls `exit_group` syscall
  - `zap_other_threads` sends `SIGKILL` to all other threads
  - `do_exit` exits the current thread
- `do_exit`
  - it disables the current thread from various subsystems
  - `group_dead` is true if this is the last alive thread
  - `disassociate_ctty` sends signals if this is a session leader
    - it sends `SIGHUP` and `SIGCONT` to the fg group
      - `SIGCONT` is in case the fg group is stopped
    - it detaches the controlling tty from the entire session
  - `exit_notify` sends signals to relatives
    - if this is the main thread, and is the last alive thread,
      `do_notify_parent` sends `SIGCHLD` to parent
  - `do_task_dead` calls `__schedule` and the task is never scheduled again
