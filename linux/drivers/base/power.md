Power Management
================

## Runtime PM

- `struct dev_pm_ops`
  - `SYSTEM_SLEEP_PM_OPS` sets callbacks for system suspend/resume
    - the traditional pm
    - to suspend/resume a device on system suspend/resume
  - `RUNTIME_PM_OPS` sets callbacks for runtime suspend/resume
    - to suspend/resume a device on depending on whether the device is in use
- <https://www.kernel.org/doc/Documentation/power/runtime_pm.txt>
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
