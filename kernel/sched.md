Scheduler
=========

## Wait Queue

- `wait_event_interruptible` marks self `TASK_INTERRUPTIBLE` and keeps calling
  `schedule` until the condition is true.  Then it sets itself back to
  `TASK_RUNNING`.  `init_wait_entry` adds a `wait_queue_entry` with
  `autoremove_wake_function` as the callback, which simply calls
  `try_to_wake_up`
- `wake_up_interruptible` calls the callback function, which calls
  `try_to_wake_up`
