Kernel Hung Task
================

## `CONFIG_DETECT_HUNG_TASK`

- `hung_task_init` starts a kthread `khungtaskd`
- `check_hung_uninterruptible_tasks` checks for hung tasks periodically
  - `timeout` is `DEFAULT_HUNG_TASK_TIMEOUT` (120) by default
  - if a thread is in `TASK_UNINTERRUPTIBLE`, `check_hung_task` checks the
    thread
    - a thread is considered hung if it has no context switch since last check
