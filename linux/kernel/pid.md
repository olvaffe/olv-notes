Kernel pid
==========

## Random Notes

- `copy_process` without `CLONE_THREAD`
  - `copy_signal`
    - `tsk->signal` points to a newly alloc `signal_struct`
  - `alloc_pid` allocs a new pid
  - `p->pid = pid_nr(pid)` sets the pid
  - `p->group_leader = p` sets the thread group leader
    - this is the thread group leader (main thread), not the process group
      leader
  - `p->tgid = p->pid` sets the thread group id
  - `p->real_parent = current` sets the parent
  - `p->exit_signal = args->exit_signal`, which defaults to `SIGCHLD`
  - `init_task_pid(p, PIDTYPE_PID, pid)`
  - `thread_group_leader` returns true (main thread)
    - `init_task_pid(p, PIDTYPE_TGID, pid)`
    - `init_task_pid(p, PIDTYPE_PGID, task_pgrp(current))`
      - it inherits pgid from parent
    - `init_task_pid(p, PIDTYPE_SID, task_session(current))`
      - it inherits sid from parent
    - `p->signal->tty = tty_kref_get(current->signal->tty)`
      - it inherits controlling terminal from parent
- `copy_process` with `CLONE_THREAD`
  - `copy_signal` is nop and `p->signal` inherits from `current->signal`
  - `alloc_pid` allocs a new pid
  - `p->pid = pid_nr(pid)` sets the pid
  - `p->group_leader = current->group_leader`
  - `p->tgid = current->tgid`
  - `p->real_parent = current->real_parent`
  - `p->exit_signal = -1`
  - `thread_group_leader` returns false
- `struct task_struct`
  - it represents a thread, not a process
    - a process is called a thread group instead
  - `p->pid` is the thread id
  - `p->group_leader` is the main thread of the thread group
  - `p->tgid` is the thread group id
  - `p->real_parent` is the parent thread group
  - `p->thread_id` is also the thread id (`struct pid`, not `pid_t`)
  - `p->signal` is shared among the thread group
    - `p->signal->leader` indicates session leader
    - `p->signal->tty` is the controlling terminal
    - `p->signal->pids` are tgid/pgid/sid
- `struct pid`
  - `pid_task` returns `pid->tasks[type]`

## Shell

- when `login` forks off shell on a tty,
  - the child calls `setsid`
    - the child is the only process and the session leader in the new session
    - the child is the only process and the group leader in the new group
  - it also calls `TIOCSCTTY`
    - the controlling terminal is the tty
  - it `exec` to the shell
- when shell forks off a job,
  - it forks off one or more children
  - the children call `setpgid(0, first-child)`
    - all children are moved to the same process group
    - the first child is the group leader
  - the children call `tcsetpgrp(tty, first-child)`
    - the process group is foreground
- `setsid` calls `ksys_setsid`
  - `group_leader->signal->leader = 1` marks the thread group leader (that is,
    the process) the session leader
  - `set_special_pids` calls `change_pid` to update
    - `task->signal->pids[PIDTYPE_SID]`
    - `task->signal->pids[PIDTYPE_PGID]`
  - `proc_clear_tty` clears `p->signal->tty`
  - upon tty hangup, such as hw removal or simulated by `vhangup`,
    `tty_vhangup` calls `tty_signal_session_leader`
    - it sends `SIGHUP` and `SIGCONT` to `tty->ctrl.session`
- `TIOCSCTTY` is handled by `tiocsctty`
  - it makes the specified tty the controlling terminal
  - `proc_set_tty` updates all of
    - `tty->ctrl.pgrp`
    - `tty->ctrl.session`
    - `current->signal->tty`
- `setpgid`
  - `change_pid` updates `PIDTYPE_PGID`
    - that is, `task->signal->pids[PIDTYPE_PGID]`
- `tcsetpgrp` is `TIOCSPGRP` and is handled by `tiocspgrp`
  - it makes the specified process group foreground by updating
    `real_tty->ctrl.pgrp`
  - upon ctrl-c, `n_tty_receive_char_special` calls
    `n_tty_receive_signal_char` to `kill_pgrp` the foreground group
  - upon normal key press, `n_tty_receive_char` calls `put_tty_queue` to add
    key to `tty->disc_data`
    - when the foreground group reads from the tty line, `tty_read` calls
      `n_tty_read` to read data from `tty->disc_data`
      - `job_control` calls `__tty_check_change` to check if
        `task_pgrp(current) == tty->ctrl.pgrp`
