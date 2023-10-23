Kernel power
============

## Initialization

- `pm_init` initializes the PM subsystem
- `/sys/power`
  - `state_attr` sets/gets the system pm state.
  - `CONFIG_PM_TRACE`
    - `pm_trace_attr` sets `pm_trace_enabled`.  I guess it remembers the last
      suspended device (for debugging).
    - `pm_trace_dev_match_attr` shows the last suspended device, I guess.
  - `CONFIG_PM_SLEEP`
    - `pm_async_attr` sets `pm_async_enabled`.  It allows devices to be
      suspended asynchronously
    - `wakeup_count_attr` is for suspend abortion
      - userspace reads ands aves the current count
      - userspace prepares for suspend
      - userspace writes the saved count
      - if the write is successful, userspace writes to `state` to suspend
    - `CONFIG_SUSPEND`
      - `mem_sleep_attr` sets `mem_sleep_current`, which can be
        - `PM_SUSPEND_TO_IDLE` aka `s2idle`
        - `PM_SUSPEND_STANDBY` aka `shallow`
        - `PM_SUSPEND_MEM` aka `deep`
      - `sync_on_suspend_attr` sets `sync_on_suspend_enabled`.  It controls fs
        sync on suspend.
    - `CONFIG_PM_AUTOSLEEP`
      - `autosleep_attr` maps to `pm_autosleep_set_state` and
        `pm_autosleep_state`
    - `CONFIG_PM_WAKELOCKS`
      - `wake_lock` maps to `pm_wake_lock` and `pm_show_wakelocks`
      - `wake_unlock` maps to `pm_wake_unlock` and `pm_show_wakelocks`
    - `CONFIG_PM_SLEEP_DEBUG`
      - `pm_test` sets `pm_test_level`.  It controls at which stage a suspend
        should be aborted, to test suspend abortion
      - `pm_print_times` is a bool and sets `pm_print_times_enabled`.  It
         controls initcall timing debug messages.
      - `pm_wakeup_irq` shows `pm_wakeup_irq`, which is the last irq that woke
        up the system
      - `pm_debug_messages` is a bool and sets `pm_debug_messages_on`.  It
        controls debug messages.
  - `CONFIG_FREEZER`
    - `pm_freeze_timeout` is in ms and sets `freeze_timeout_msecs`.  It
      controls how long `freeze_processes` can run
- `pm_states` and `mem_sleep_states`
  - there are 5 numerical states
    - `PM_SUSPEND_ON` is 0 and means system on
    - `PM_SUSPEND_TO_IDLE` is 1 and means suspend-to-idle
      - `freeze` in `pm_states` and `s2idle` in `mem_sleep_states`
    - `PM_SUSPEND_STANDBY` is 2
      - `standby` in `pm_states` and `shallow` in `mem_sleep_states`
    - `PM_SUSPEND_MEM` is 3
      - `mem` in `pm_states` and `deep` in `mem_sleep_states`
    - `PM_SUSPEND_MAX` is 4
      - this is hibernation
      - aka `disk`
    - `PM_SUSPEND_MIN` is `PM_SUSPEND_TO_IDLE`
  - when writing to `/sys/power/state`,
    - if `disk`, it calls `hibernate()`
    - otherwise, it calls `pm_suspend()`
      - if `mem`, it remaps using `/sys/power/mem_sleep` first
