Device Power Management
=======================

## `struct dev_pm_ops`

- <https://docs.kernel.org/driver-api/pm/types.html>
- there are 23 ops
- init/cleanup
  - `prepare`
  - `complete`
- `SYSTEM_SLEEP_PM_OPS`, `LATE_SYSTEM_SLEEP_PM_OPS`, and
  `NOIRQ_SYSTEM_SLEEP_PM_OPS`
  - these depend on `CONFIG_PM_SLEEP`
  - there are 6 ops, with early/late and noirq variants
  - `suspend`
  - `resume`
  - `free`
  - `thaw`
  - `poweroff`
  - `restore`
- `RUNTIME_PM_OPS`
  - these depend on `CONFIG_PM`
  - `runtime_suspend`
  - `runtime_resume`
  - `runtime_idle`
- suspend / resume
  - `prepare`
  - `suspend`
  - `suspend_late`
  - `suspend_noirq`
  - `resume_noirq`
  - `resume_early`
  - `resume`
  - `complete`
- hibernation
  - `prepare`
  - `freeze`
  - `freeze_late`
  - `freeze_noirq`
  - at this point, everything is stable and a system image can be created
  - `thaw_noirq`
  - `thaw_early`
  - `thaw`
  - `complete`
  - at this point, the system image can be saved
  - `prepare`
  - `poweroff`
  - `poweroff_late`
  - `poweroff_noirq`
  - at this point, the system can be powered off/on
  - `prepare`
  - `freeze`
  - `freeze_late`
  - `freeze_noirq`
  - `restore_noirq`
  - `restore_early`
  - `restore`
  - `complete`
- `DEFINE_RUNTIME_DEV_PM_OPS` inits all ops from runtime suspend/resume/idle
  - `SYSTEM_SLEEP_PM_OPS` uses `pm_runtime_force_suspend` and
    `pm_runtime_force_resume` for everything
  - no `LATE_SYSTEM_SLEEP_PM_OPS`
  - no `NOIRQ_SYSTEM_SLEEP_PM_OPS`
  - `RUNTIME_PM_OPS` is explicitly specified

## System Suspend and Resume

- `pm_suspend` suspends the system
  - when it returns, the system has resumed
- system suspend
  - `dpm_suspend_start(PMSG_SUSPEND)` suspends the devices
    - `dpm_prepare` prepares devices on `dpm_list` and moves them to
      `dpm_prepared_list`
      - `device_prepare` calls `dev->driver->pm->prepare`
    - `dpm_suspend` suspends devices on `dpm_prepared_list` and moves them to
      `dpm_suspended_list`
      - `device_suspend` calls `dev->driver->pm->suspend`
  - `dpm_suspend_late(PMSG_SUSPEND)` late-suspends devices on
    `dpm_suspended_list` and moves them to `dpm_late_early_list`
    - `device_suspend_late` calls `dev->driver->pm->ops->suspend_late`
  - `dpm_suspend_noirq(PMSG_SUSPEND)` noirq-suspends devices on
    `dpm_late_early_list` and moves them to `dpm_noirq_list`
    - `device_wakeup_arm_wake_irqs` arms the wake irqs to wake up the system
    - `suspend_device_irqs` disables devices irqs
    - `device_suspend_noirq` calls `dev->driver->pm->ops->suspend_noirq`
- system resume
  - `dpm_resume_noirq(PMSG_RESUME)` noreq-resumes devices on `dpm_noirq_list`
    and moves them to `dpm_late_early_list`
    - `device_resume_noirq` calls `dev->driver->pm->ops->resume_noirq`
    - `resume_device_irqs` enables devices irqs
    - `device_wakeup_disarm_wake_irqs` disarms the wake irqs
  - `dpm_resume_early(PMSG_RESUME)` late-resumes devices on
    `dpm_late_early_list` and moves them to `dpm_suspended_list`
    - `device_resume_early` calls `dev->driver->pm->ops->resume_early`
  - `dpm_resume_end(PMSG_RESUME)` resumes the devices
    - `dpm_resume` resumes devices on `dpm_suspended_list` and
      moves them to `dpm_prepared_list`
      - `device_resume` calls `dev->driver->pm->ops->resume`
    - `dpm_complete` completes devices on `dpm_prepared_list` and
      moves them to `dpm_list`
      - `device_complete` calls `dev->driver->pm->ops->complete`

## Wakeup

- device driver
  - if the hw is capable of system wakeup, the driver calls
    `device_set_wakeup_capable`
  - if the hw should wake up the device (e.g., hid keyboard), the driver calls
    `device_set_wakeup_enable`
    - `wakeup_source_register` registers a wakeup source
    - `device_wakeup_attach` points `ws->wakeirq` to `dev->power.wakeirq`
- on suspend, `suspend_enter` calls `device_wakeup_arm_wake_irqs` indirectly
  - it loops through all wakeup sources
  - `irq_set_irq_wake` enables the irq to wake up the system
- on resume, `suspend_enter` continues to call
  `device_wakeup_disarm_wake_irqs` indirectly
  - it loops through all wakeup sources
  - `irq_set_irq_wake` disables the irq

## Runtime PM

- `struct dev_pm_ops`
  - `SYSTEM_SLEEP_PM_OPS` sets callbacks for system suspend/resume
    - the traditional pm
    - to suspend/resume a device on system suspend/resume
  - `RUNTIME_PM_OPS` sets callbacks for runtime suspend/resume
    - to suspend/resume a device on depending on whether the device is in use
- <https://docs.kernel.org/power/runtime_pm.html>
  - `pm_runtime_resume` and `pm_request_resume` resumes a device
    - `pm_runtime_resume` resumes the device before returning
    - `pm_request_resume` just queues a resume request
  - `pm_runtime_suspend` and `pm_schedule_suspend` suspends a device
    - `pm_runtime_suspend` suspends the device before returning
    - `pm_schedule_suspend` just queues a suspend request
  - more commonly, use `pm_runtime_get_sync` and `pm_runtime_put_autosuspend`
    - `pm_runtime_get_sync` increases the usage count and makes sure the
      device is resumed before returning
    - `pm_runtime_put_autosuspend` decreases the usage count and queues an
      autosuspend request
  - autosuspend
    - like suspend, except there is a delay that is configurable by the driver
      or the userspace
    - `/sys/devices/.../power/autosuspend_delay_ms`, -1 to disable autosuspend
  - e.g., msm uses autosuspend
    - on init, sets the autosuspend delay to `DRM_MSM_INACTIVE_PERIOD` (66ms)
    - on submit, if the submit queue becomes non-empty, `pm_runtime_get`
    - on retire, if the submit queue becomes empty,
      `pm_runtime_put_autosuspend`
      - also calls `pm_runtime_mark_last_busy`
- driver writer
  - runtime pm assumes the device is suspended by default
    - use `pm_runtime_set_active` if the assumption is incorrect
  - during probe,
    - `devm_pm_runtime_enable` enables runtime pm
    - `pm_runtime_use_autosuspend` enables autosuspend
      - `pm_runtime_set_autosuspend_delay` sets a delay
  - using the device,
    - `pm_runtime_resume_and_get` resumes the dev
    - `pm_runtime_mark_last_busy` updates the access time for autosuspend
    - `pm_runtime_put_autosuspend` suspends the dev
