Linux VFS Poll
==============

## Overview

- `poll -> do_sys_poll -> do_poll` is infinite loop
  - it calls `do_pollfd` on each fd
    - `f_op->poll` calls `poll_wait` such that poll can add a wait entry to
      the impl's wait queue
  - it sets `pt->_qproc` to `NULL`
    - future `poll_wait` calls become nop
  - it exits the loop only when
    - events
    - timeout
    - signal
  - `poll_schedule_timeout` sleeps until
    - impls wake up their wait queues
    - timeout
    - signal
