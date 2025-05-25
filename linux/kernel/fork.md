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
    - `alloc_thread_stack_node` allocs `tsk->stack`
      - this is the kernel stack whose size is `THREAD_SIZE`
    - `set_task_stack_end_magic` sets `end_of_stack` to `STACK_END_MAGIC`
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

## PID and TID

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

## `mm_struct`

- `mm_struct` is created in two ways
  - when a task is forked, `copy_mm` calls `dup_mm` to duplicate from the
    original task
  - when exec, `bprm_mm_init` calls `mm_alloc` to allocate a new one
- `mm_init` initializes a newly-allocated `mm_struct`
  - `mm_alloc_pgd` calls `pgd_alloc` to allocate `pgd_t` array
    - both `pgd_alloc` and `pgd_t` are arch-specific
      - `pgd_t` is commonly a struct with a u64 inside
        - the u64 is a entry descriptor
      - the allocation is often a page, `__get_free_page`
        - array size is thus `4096/8=512`
  - `mm_alloc_pgd` calls `pgd_alloc` to allocate `pgd_t` array
- `__bprm_mm_init` inserts the first vma, the stack, to the mm with
  `vm_area_alloc` and `insert_vm_struct`

## Kernel Stack

- modern archs have `CONFIG_THREAD_INFO_IN_TASK`
- each task has a kernel stack, `task->stack`, whose size is `THREAD_SIZE`
- `linux/sched/task_stack.h`
  - `task_stack_page` retruns `task->stack`
  - `setup_thread_stack` is nop
  - `end_of_stack` also returns `task->stack`
  - `task_stack_end_corrupted` tests if `end_of_stack` contains
    `STACK_END_MAGIC`
  - `object_is_on_stack` returns true if the obj is in kernel stack
- `linux/thread_info.h`
  - `current_thread_info` returns `current`, because the first member of
    `task_struct` is `thread_info`
  - `thread_info` is arch-specific and has at least
    - `flags`, for use with - `*_ti_thread_flag`
    - `cpu`, for `set_task_cpu` and `task_cpu`
- x86
  - kernel stack layout, from top to bottom
    - `TOP_OF_KERNEL_STACK_PADDING`
    - `pt_regs`
    - stack frames, initially empty
    - `inactive_task_frame`
  - accessors
    - `task_top_of_stack` returns addr of `TOP_OF_KERNEL_STACK_PADDING`
    - `task_pt_regs` returns addr of `pt_regs`
  - `copy_thread` inits kernel stack
    - `inactive_task_frame`
      - `ret_addr` is `ret_from_fork_asm`
    - `p->thread.sp` points to `inactive_task_frame`
    - `pt_regs` is copied from `current`
      - `*childregs = *current_pt_regs();`
      - `childregs->ax = 0;`
        - because `fork` returns 0 for child
  - `switch_to` defines to `__switch_to_asm`
    - it pushes `inactive_task_frame` onto `prev` stack
      - the caller has pushed `ret_addr`
    - it switches stacks
      - it copies `rsp` to `prev->thread.sp`
      - it copies `next->thread.sp` to `rsp`
    - it pops `inactive_task_frame` from `next` stack
      - except for `ret_addr`
    - it jumps to `__switch_to`
      - `current_task` points to `next_p`
      - `cpu_current_top_of_stack` points to `task_top_of_stack(next_p)`
      - return pops `ret_addr` and jumps to it
  - when scheduler `switch_to` to a new task with the new kernel stack, it
    returns to `ret_addr` which points to `ret_from_fork_asm`
    - it jumps to `swapgs_restore_regs_and_return_to_usermode` and follows the
      same return path as `common_interrupt_return`
    - `POP_REGS` restores regs saved by `PUSH_AND_CLEAR_REGS`, which are part
      of `pt_regs`
    - it skips `orig_ax`
    - `iretq` pops the rest of `pt_regs` automatically
