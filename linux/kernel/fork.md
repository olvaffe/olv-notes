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

## namespaces

- `clone` is similiar to `fork` and has more controls
  - e.g., setting `CLONE_THREAD` creates a thread
- `unshare` allows a process (or thread) to control its shared execution
  context without creating a new process (or thread)
- interesting namespace-related flags are
  - `CLONE_NEWCGROUP`
    - new cgroup namespace
    - require root
  - `CLONE_NEWIPC`
    - new ipc namespace for `sysvipc` and `mq_overview`
    - require root
  - `CLONE_NEWNET`
    - new net namespace
    - require root
    - initially only `lo`
    - each net device can only belong to a single net namespace
      - use `ip netns` to manage
  - `CLONE_NEWNS`
    - new mount namespace
    - require root
    - initial values are inherited from parent
  - `CLONE_NEWPID`
    - new pid namespace
    - require root
    - when used with `unshare`, only affects child processes created
      afterwards
      - the first child will have pid 1
    - `/proc` shows the pid namespace of the process who mounts it
      - normally, pid 1 should chroot and mount `/proc`
      - or, pid 1 can mount over `/proc` with new mount namespace
      - need user namespace to be able to mount
  - `CLONE_NEWTIME`
    - new time namespace for system clocks such as `CLOCK_MONOTONIC`
    - require root
    - can only be set with `unshare` and only affects child processes created
      afterwards
  - `CLONE_NEWUSER`
    - new user namespace for uids and gids
    - no priviledge required
    - until uid/gid mappings are set up, the process has unmapped uid/gid
      which gets overflowed to 65534
    - all capabilities are granted
      - note that `execve` will drop capabilities unless the uid is 0
  - `CLONE_NEWUTS`
    - new UTS namespace for host name and (unused) NIS domain name
    - root-only
    - initial values are inherited from parent
    - UTS stands for unix time sharing and comes from `utsname` of `uname`
