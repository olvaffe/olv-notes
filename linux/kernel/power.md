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

## Suspend

- `pm_suspend` is the entrypoint
  - triggered by `echo foo > /sys/power/state`
  - it supports 3 states
    - `PM_SUSPEND_TO_IDLE`
    - `PM_SUSPEND_STANDBY`
    - `PM_SUSPEND_MEM`
- flow
  - `s2idle_begin` if `PM_SUSPEND_TO_IDLE`
    - this sets `s2idle_state = S2IDLE_STATE_NONE`
  - `ksys_sync_helper` if `sync_on_suspend_enabled` (the default)
    - this calls `ksys_sync` (the `sync` syscall)
    - `Filesystems sync: %ld.%03ld seconds`
  - `Preparing system for sleep (%s)`
    - `%s` is `s2idle` (idle), `shallow` (standby), or `deep` (mem)
  - `suspend_prepare`
    - `pm_notifier_call_chain_robust(PM_SUSPEND_PREPARE, PM_POST_SUSPEND)`
    - `suspend_freeze_processes`
      - `freeze_processes` freezes all processes
      - `freeze_kernel_threads` freezes all kernel threads
      - `Freezing %s`
        - `%s` is `user space processes` or `remaining freezable tasks`
    - `pm_notifier_call_chain(PM_POST_SUSPEND)`
  - `suspend_test(TEST_FREEZER)`
    - it causes the suspend abort if `echo freezer > /sys/power/pm_test`
    - that is, abort after processes are freezed
  - `Suspending system (%s)`
  - `suspend_devices_and_enter`
    - `pm_suspend_target_state` is set to the target state
    - `platform_suspend_begin`
      - on x86, this calls `acpi_s2idle_begin` or `acpi_suspend_begin`
        - `PM_SUSPEND_ON` maps to `ACPI_STATE_S0`
        - `PM_SUSPEND_STANDBY` maps to `ACPI_STATE_S1`
        - `PM_SUSPEND_MEM` maps to `ACPI_STATE_S3`
        - `PM_SUSPEND_MAX` maps to `ACPI_STATE_S5`
    - `dpm_suspend_start(PMSG_SUSPEND)`
      - `dpm_prepare` prepares devices for PM transition
      - `dpm_suspend` suspends devices
    - `suspend_test(TEST_DEVICES)`
      - it causes suspend abort if `echo devices > /sys/power/pm_test`
    - `suspend_enter`
      - `platform_suspend_prepare`
        - on x86, this is only for s2idle
      - `dpm_suspend_late(PMSG_SUSPEND)` calls `device_suspend_late` on all
        deivces
      - `platform_suspend_prepare_late`
        - on x86, this is only for s2idle
      - `dpm_suspend_noirq(PMSG_SUSPEND)` disables device irqs and calls
        `device_suspend_noirq`
      - `platform_suspend_prepare_noirq`
        - on x86, this is only for suspend and calls `acpi_pm_prepare`
      - `suspend_test(TEST_PLATFORM)`
        - it causes suspend abort if `echo platform > /sys/power/pm_test`
      - `s2idle_loop` if s2idle
        - `suspend-to-idle`
        - `s2idle_enter` loop
        - `resume from suspend-to-idle`
      - `pm_sleep_disable_secondary_cpus`
      - `suspend_test(TEST_CPUS)`
        - it causes suspend abort if `echo processors > /sys/power/pm_test`
      - `arch_suspend_disable_irqs`
      - `syscore_suspend`
      - `suspend_test(TEST_CORE)`
        - it causes suspend abort if `echo core > /sys/power/pm_test`
      - `suspend_ops->enter`
      - on resume,
        - `syscore_resume`
        - `arch_suspend_enable_irqs`
        - `pm_sleep_enable_secondary_cpus`
        - `platform_resume_noirq`
        - `dpm_resume_noirq`
        - `platform_resume_early`
        - `dpm_resume_early`
        - `platform_resume_finish`
    - `dpm_resume_end(PMSG_RESUME)`
      - `dpm_resume` resumes devices
      - `dpm_complete` completes the PM transition
    - `platform_resume_end`
  - `Finishing wakeup.`
  - `suspend_finish`
    - `suspend_thaw_processes`
    - `pm_notifier_call_chain(PM_POST_SUSPEND)`
