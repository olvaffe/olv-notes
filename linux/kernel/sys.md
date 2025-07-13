Kernel sys
==========

## UTS name

- UTS stands for UNIX Time-Sharing
- `struct new_utsname`
  - `utsname` returns the `new_utsname` of the current task
    - `&current->nsproxy->uts_ns->name`
- `init_task` is a statically-initialized task with pid 0
  - `nsproxy` is `init_nsproxy`
  - `uts_ns` is `init_uts_ns`
    - `sysname` is `Linux`
    - `nodename` is `(none)`
      - this is the host name
      - it can be customized by `CONFIG_DEFAULT_HOSTNAME` or kernel cmdline
    - `release` is something like `6.2.2+`
      - it is generated at `include/generated/utsrelease.h` and can be
        customized
    - `version` is something like `#34 SMP PREEMPT_DYNAMIC Wed Mar  8 15:24:05 PST 2023`
      - it is generated at `include/generated/utsversion.h` and can be
        customized
    - `machine` is something like `x86_64`
      - it is defined by arch
    - `domainname` is `(none)`
      - this is NIS domain name and is irrelevant on modern systems
  - incidentally, pid 1 and pid 2 are forked from pid 0
    - as can be seen by `ps -ef`
    - they inherit the uts name from pid 0
    - pid 1 runs `kernel_init` and execves `/sbin/init` at the end
    - pid 2 runs `kthreadd`
- syscalls
  - `newuname` copies `struct new_utsname` to userspace buf
  - `uname` copies `struct old_utsname` to userspace buf and is not used by
    userspace anymore
  - `olduname` copies `struct oldold_utsname` to userspace buf and is not
    used by userspace anymore
  - `sethostname` updates `nodename`
    - the length cannot exceed `__NEW_UTS_LEN` (64)
    - on boot, userspace parses `/etc/hostname` and calls `sethostname`
    - systemd recommends to not include dots
  - `gethostname` copies `nodename` to userspace buf and is not used by
    userspace anymore
  - `setdomainname` updates `domainname` and is not used by userspace anymore

## `prctl`

- `PR_SET_NAME` sets the thread name
  - `set_task_comm` copies the name to `task->comm[TASK_COMM_LEN]`
  - `TASK_COMM_LEN` is 16

## System calls for process control

- After a process is `fork()`ed, it is in the same session and process group as
  its parent is.
- `setsid()` creates a new session and a new process group with the calling
  process as the leader of the session and the process group.  The IDs of the
  session and the process group are the same as the ID of the calling process,
  making the process both the session and process group leader.  The calling
  process will be the only process in the new session and process group.
  - the new session has no controlling terminal
  - a group leader cannot call `setsid()`
- `setpgid()` moves a process to a process group.  It can only be called on the
  calling process itself or the childen of the calling process.
  - It is used by a shell to implement job control.
- `waitpid()` waits until the specified process exits or is stopped by a signal
- `open()`ing a tty device makes the tty device the controlling terminal of the
  calling process if there isn't one yet.
  - to avoid this behavior, `O_NOCTTY` should be specified
  - there is an undocumented `TIOCSCTTY` ioctl to set the controlling terminal
  - `TIOCNOTTY` ioctl can detach the calling process from its controlling
    terminal.  It is used by daemons.  See `tty(4)`.
- `tcsetpgrp()` sets the foreground process group id of a terminal.
  - Only processes in the foreground process group can receive input from the
    controlling terminal, including signals such as `SIGINT`.
  - A background process group trying to read from the terminal receives
    `SIGTSTP`.
  - When the controlling terminal hangs up or the session leader detaches from
    its controlling terminal, `SIGHUP` and `SIGCONT` are sent to the foreground
    process group.
  - A background process group calling `tcsetpgrp()` will receive `SIGTTOU`
    unless the calling process blocks or ignores the signal.

## System calls for process control (kernel)

- "pid", "tid", and "task" are used interchangeably and refer to a
  `struct task_struct` in kernel
  - That is, a thread as known in the user space
- "tgid", "process", and "thread group" are used interchangeably and refer to
  tasks that share an `struct mm_struct` in kernel
  - That is, a process as known in the user space
- A `struct pid` is a process identifier referring to tasks, process groups, and
  sessions.
  - it is also namespaced
  - a `pid` knows the tasks, process groups, and sessions using it
- The pid of a task is `task->thread_pid`.
- The tgid/pgid/sid of a task are
  - `task->signal->pids[PIDTYPE_TGID]`
  - `task->signal->pids[PIDTYPE_PGID]`
  - `task->signal->pids[PIDTYPE_SID]`
    `task->group_leader->pids[PIDTYPE_SID].pid`
- Just remember that the group leader of a task is the main thread of a
  process in the user space
- A thread is a thread group leader (main thread) if
  - `task->group_leader == task`
- `task->real_parent` is who forks the task
- A task is a process group leader if there is a process group with the same
  PGID as the thread group leader's pid
  - that is, `pid_task(task->group_leader, PIDTYPE_PGID)` returns something
- A session leader is a task with `task->group_leader->signal->leader` set to 1
- The controlling tty of a task is `task->signal->tty`
- The controlled session of a tty is `tty->ctrl.session`
  - Given a task and its controlling tty, we have
    `task_session(task) == tty->ctrl.session` and
    `task->group_leader->signal->tty == tty`
- The foreground process group of a tty is `tty->ctrl.pgrp`

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
