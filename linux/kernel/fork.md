Kernel fork
===========

## `kernel_clone`

- `kernel_clone` was named `_do_fork` and is how `struct task_struct` is
  allocated.  It only has a few callers
  - `kernel_thread`
  - `user_mode_thread`
  - `fork`, `vfork`, `clone`, and `clone3` syscalls
- `copy_process` deep-copies `current`
  - it copies A LOT; some interesting ones are...
  - `dup_task_struct` duplicates `struct task_struct`
    - this allocates the stack for the new task as well
  - `sched_fork` re-initializes the fields used by the scheduler
  - `copy_files` dups `struct files_struct`, for opened files
  - `copy_fs` dups `struct fs_struct`
  - `copy_sighand` dups `struct sighand_struct`, signal handlers
  - `copy_signal` dups `struct signal_struct`
  - `copy_mm` dups `struct mm_struct`
  - `copy_thread` dups the thread
    - this is arch-dependent and dups saved registers
  - `alloc_pid` allocs a new `struct pid`
  - `init_task_pid` sets the pid pointer
- `wake_up_new_task`
  - marks the task `TASK_RUNNING`
  - calls `select_task_rq` to select the cpu for the task
    - `struct rq` is runqueue
  - `activate_task` adds the task to the cpu runqueue
    - `sched_core_enabled` is normally false; it was motivated mostly by cpu
      smt security issues
