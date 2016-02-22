Terminals and Processes
=======================

## Overview

* the application runs on the host with linux kernel
* a termial is connected to the host for I/O with a serial line
* the character set over the serial line is ASCII (7- or 8-bit)
* terminal as well as the serial line can be virtual
  * `pts` provides virtual lines
    * `gnome-terminal` uses `pts` to function as a terminal
    * An SSH connection uses `pts` on the remote (host)
  * linux VTs are virtual terminals and lines
* `termios` (or the linux-specific) `tty_ioctl` interface can be used to control
  the serial line on the host
* escape sequences are used to control the terminal from the host
  * Linux-specific(?) `console_ioctl` interface can be used to control the VT
* devices
  * `/dev/ttyS[0..3]` COM port serial line
  * `/dev/tty[0..64]` VT serial lines and terminals
  * `/dev/tty0` current VT
  * `/dev/tty` the controlling terminal
  * `/dev/console` kernel console (where kernel messages go to)

## IDs of a process

* Example

    olvaffe@m500:~$ tail -f /var/log/messages | grep aaa &
    [1] 11977
    olvaffe@m500:~$ ps -o tty,comm,uid,gid,sess,pgid,pid,ppid
    TT       COMMAND           UID   GID  SESS  PGID   PID  PPID
    pts/4    bash             1000  1000 11971 11971 11971  4628
    pts/4    tail             1000  1000 11971 11976 11976 11971
    pts/4    grep             1000  1000 11971 11976 11977 11971
    pts/4    ps               1000  1000 11971 11988 11988 11971

* This new shell is in a new session, 11971
* A session can have a controlling terminal (or not), pts/4 in this case
* There are several process groups, one for each command line execution
* At any time, at most one of the process groups in the session can be the
  foreground process group.
* The foreground process groups receives inputs and signals from the controlling
  terminal.

## System calls for process control

* After a process is `fork()`ed, it is in the same session and process group as
  its parent is.
* `setsid()` creates a new session and a new process group with the calling
  process as the leader of the session and the process group.  The IDs of the
  session and the process group are the same as the ID of the calling process,
  making the process both the session and process group leader.  The calling
  process will be the only process in the new session and process group.
  * the new session has no controlling terminal
  * a group leader cannot call `setsid()`
* `setpgid()` moves a process to a process group.  It can only be called on the
  calling process itself or the childen of the calling process.
  * It is used by a shell to implement job control.
* `waitpid()` waits until the specified process exits or is stopped by a signal
* `open()`ing a tty device makes the tty device the controlling terminal of the
  calling process if there isn't one yet.
  * to avoid this behavior, `O_NOCTTY` should be specified
  * there is an undocumented `TIOCSCTTY` ioctl to set the controlling terminal
  * `TIOCNOTTY` ioctl can detach the calling process from its controlling
    terminal.  It is used by daemons.  See `tty(4)`.
* `tcsetpgrp()` sets the foreground process group id of a terminal.
  * Only processes in the foreground process group can receive input from the
    controlling terminal, including signals such as `SIGINT`.
  * A background process group trying to read from the terminal receives
    `SIGTSTP`.
  * When the controlling terminal hangs up or the session leader detaches from
    its controlling terminal, `SIGHUP` and `SIGCONT` are sent to the foreground
    process group.
  * A background process group calling `tcsetpgrp()` will receive `SIGTTOU`
    unless the calling process blocks or ignores the signal.

## System calls for process control (kernel)

* see `kernel/sys.c`
* "pid", "tid", and "task" are used interchangeably and refer to a
  `struct task_struct` in kernel
  * That is, a thread as known in the user space
* "tgid", "process", and "thread group" are used interchangeably and refer to
  tasks that share an `struct mm_struct` in kernel
  * That is, a process as known in the user space
* A `struct pid` is a process identifier referring to tasks, process groups, and
  sessions.
  * it is also namespaced (for virtualization?)
  * a `pid` knows the tasks, process groups, and sessions using it
* The pid of a task is `task->pids[PIDTYPE_PID].pid`.  The tgid of a task is
  thus `task->group_leader->pids[PIDTYPE_PID].pid`
  * and the pgid and sid are `task->group_leader->pids[PIDTYPE_PGID].pid` and
    `task->group_leader->pids[PIDTYPE_SID].pid`
  * just remember that the group leader of a task is the main thread of a
    process in the user space
* A thread is a thread group leader if `task->group_leader == task`
* `task->real_parent` is who forks the task
* A task is a process group leader if there is a process group with the same
  PGID as the thread group leader's pid
  * that is, `pid_task(task->group_leader, PIDTYPE_PGID)` returns something
* A session leader is a task with `task->group_leader->signal->leader` set to 1
* The controlling tty of a task is `task->signal->tty`
* The controlled session of a tty is `tty->session`
  * Given a task and its controlling tty, we have
    `task_session(task) == tty->session` and
    `task->group_leader->signal->tty == tty`
* The foreground process group of a tty is `tty->pgrp`

## fbdev

* In kernel, a fbdev is a VT driver.  That is how VT draws characters on screen.
* The first thing to do is to switch to a new VT and ask the VT not to draw to
  the screen
  * `ioctl(fd, VT_ACTIVATE, vtno)`
  * `ioctl(fd, KDSETMODE, KD_GRAPHICS)`
* Then we can open the fbdev device and draw at will
* Xorg
  * observed via `ps -ax -o sid,comm,tpgid,pid,tty` and the source
  * The server is in its own session.
  * It has the new VT as the controlling terminal
    * to set the new VT to graphics mode
    * to avoid other processes stealing the VT
  * It is also the foreground process of the new VT
  * Input events are received from evdev
    * the new VT is constantly flushed to keep its buffer clean
    * the new VT is made raw to avoid signals and others
  * It is also the foreground process of the old VT
    * it can receive Ctrl-C from the old VT
