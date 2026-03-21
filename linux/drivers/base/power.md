Device Power Management
=======================

## `struct dev_pm_ops`

- <https://docs.kernel.org/driver-api/pm/types.html>
- there are 23 ops
  - init/cleanup
    - `prepare`
    - `complete`
  - system suspend/resume/hibernate
    - these depend on `CONFIG_PM_SLEEP`
    - there are 6 ops, with early/late/noirq variants
      - `suspend`
      - `resume`
      - `freeze`
      - `thaw`
      - `poweroff`
      - `restore`
  - runtime suspend/resume
    - these depend on `CONFIG_PM`
    - there are 3 ops
      - `runtime_suspend`
      - `runtime_resume`
      - `runtime_idle`
- `suspend_devices_and_enter` executes system suspend and resume
  - `prepare`
  - `suspend`
  - `suspend_late`
  - `suspend_noirq`
  - system suspended
  - system resumed
  - `resume_noirq`
  - `resume_early`
  - `resume`
  - `complete`
- `hibernation_snapshot` executes system image snapshot
  - `prepare`
  - `freeze`
  - `freeze_late`
  - `freeze_noirq`
  - at this point, everything is stable and a system image is created
  - `thaw_noirq`
  - `thaw_early`
  - `thaw`
  - `complete`
- `power_down` executes system hibernation
  - if `hibernation_platform_enter`,
    - `prepare`
    - `poweroff`
    - `poweroff_late`
    - `poweroff_noirq`
  - if `kernel_power_off`,
    - `dev->driver->shutdown` instead of `dev_pm_ops`
  - at this point, the system is powered off
- `hibernation_restore` executes system image restore
  - `prepare`
  - `freeze`
  - `freeze_late`
  - `freeze_noirq`
  - at this point, everything is stable and a system image is restored
  - it continues from where the system image was created in `swsusp_arch_suspend`
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

## System Wakeup

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

## Runtime PM Internals

- `rpm_resume` resumes a device
  - `dev->power.request` is set to `RPM_REQ_NONE` to cancel async op
  - if `dev->power.runtime_status` is already `RPM_ACTIVE`, return 1
  - if another `rpm_resume` or `rpm_suspend` is in-flight,
    - if `RPM_NOWAIT`, return `-EINPROGRESS`
    - it waits until the another op has finished
  - if `RPM_ASYNC`,
    - `dev->power.request` is set to `RPM_REQ_RESUME`
    - `dev->power.request_pending` is set
    - `pm_runtime_work` is scheduled
    - return 0
  - if parent,
    - `rpm_resume` resumes parents recursively
  - `dev->power.runtime_status` is set to `RPM_RESUMING`
  - `rpm_callback` calls the callback
    - `rpm_get_suppliers` resumes all suppliers first
  - `dev->power.runtime_status` is set to `RPM_ACTIVE`
  - `rpm_idle(RPM_ASYNC)` suspends the device again async
    - it will fail if the usage is non-zero
- `rpm_suspend` suspends a device
  - `rpm_check_suspend_allowed` requires the usage to be zero and all children
    are suspended
  - if another `rpm_resume` is in-flight, and this is not `RPM_ASYNC`, return
    `-EAGAIN`
  - if `RPM_AUTO` and no other `rpm_suspend` in-flight,
    - `pm_runtime_autosuspend_expiration` returns when the auto-suspend should
      take place
    - if non-zero, it takes place later
      - `dev->power.timer_expires` is updated
      - `dev->power.suspend_timer` is armed
      - `dev->power.timer_autosuspends` is set
      - return 0
  - `pm_runtime_cancel_pending` cancels pending op
  - if another `rpm_suspend` is in-flight,
    - if `RPM_ASYNC` or `RPM_NOWAIT`, return `-EINPROGRESS`
    - it waits until the another op has finished
  - if `RPM_ASYNC`,
    - `dev->power.request` is set to `RPM_REQ_AUTOSUSPEND` or `RPM_REQ_SUSPEND`
    - `dev->power.request_pending` is set
    - `pm_runtime_work` is scheduled
    - return 0
  - `dev->power.runtime_status` is set to `RPM_SUSPENDING`
  - `rpm_callback` calls the callback
    - `__rpm_put_suppliers` drops usage counts of suppliers last
  - `dev->power.runtime_status` is set to `RPM_SUSPENDED`
  - if `dev->power.deferred_resume`,
    - it means another `rpm_resume` defers its op to here
    - call `rpm_resume` and return `-EAGAIN`
  - if parent,
    - `rpm_idle(parent, RPM_ASYNC)` puts the parent to idle async
  - `rpm_suspend_suppliers` puts supplier to idle async
- `rpm_idle` auto-suspends a device after idle check and notification
  - `rpm_check_suspend_allowed` requires the usage to be zero and all children
    are suspended
  - `dev->power.runtime_status` remains at `RPM_ACTIVE`
    - if `RPM_GET_PUT`, it can remain at `RPM_SUSPENDED` and this is nop
  - if `RPM_ASYNC`,
    - `dev->power.request` is set to `RPM_REQ_IDLE`
    - `dev->power.request_pending` is set
    - `pm_runtime_work` is scheduled
    - return 0
  - it calls `runtime_idle` callback
    - the callback returns an error if the device is not really idle, which
      causes the suspend to be skipped
    - this is named "idle notification" somehow
  - unless already suspended or error, `rpm_suspend(RPM_AUTO)` auto-suspends
- `pm_runtime_work` executes `dev->power.request` async
  - it calls one of `rpm_idle`, `rpm_suspend`, and `rpm_resume`

## Runtime Scheduled Suspend and Auto Suspend

- `pm_schedule_suspend` schedules a suspend
  - if no delay, it simply calls `rpm_suspend(RPM_ASYNC)`
  - `rpm_check_suspend_allowed` checks if suspend is allowed
  - `pm_runtime_cancel_pending` cancels pending op
  - `dev->power.timer_expires` is set to the abs ts
  - `dev->power.timer_autosuspends` is cleared, indicating the timer is used
    for scheduled suspend
  - `dev->power.suspend_timer` is started
- when `pm_suspend_timer_fn` fires,
  - `dev->power.timer_expires` is set to 0
  - `rpm_suspend(RPM_ASYNC)` suspends the device
    - if auto-suspend, `RPM_AUTO` as well
- `pm_runtime_set_autosuspend_delay` and `pm_runtime_use_autosuspend` enable
  auto-suspend
  - if delay is negative, it holds a usage count and the device is resumed
  - else, `rpm_idle(RPM_AUTO)` attempts to auto-suspend
    - if delay is 0, `pm_runtime_autosuspend_expiration` always returns 0 and
      the suspend goes through
    - else, if the auto-suspend should be delayed, `rpm_suspend` arms the
      hrtimer

## Runtime PM APIs

- <https://docs.kernel.org/power/runtime_pm.html>
- `device_pm_init` inits pm and rpm for a device
  - `device_pm_init_common` inits common fields in `dev->power`
  - `device_pm_sleep_init` inits pm fields in `dev->power`
  - `pm_runtime_init` inits rpm fields in `dev->power`
    - `dev->power.runtime_status` is `RPM_SUSPENDED`
    - `dev->power.last_status` is `RPM_INVALID`
    - `dev->power.disable_depth` is 1
    - the rest is default/zero
- `pm_runtime_enable` and `pm_runtime_disable` enable/disable rpm for a device
  - resume/suspend will return `-EACCES` if `dev->power.disable_depth`
  - they are called by drivers when rpm callbacks are ready
- `pm_runtime_block_if_disabled` and `pm_runtime_unblock` block/unblock rpm
  enable
  - `pm_runtime_enable` will result in a warning
  - they are called from system suspend and resume respectively, to prevent
    unexpected runtime enable during suspend
- `pm_runtime_get_if_active` and `pm_runtime_get_if_in_use` keep active
  device active by incrementing usage
  - `pm_runtime_get_if_active` checks for `RPM_ACTIVE`
  - `pm_runtime_get_if_in_use` checks for both `RPM_ACTIVE` and current usage
  - during autosuspend delay, the device is `RPM_ACTIVE` but usage is 0
- `pm_runtime_forbid` and `pm_runtime_allow` are similar to
  `pm_runtime_get_sync` and `__pm_runtime_put_autosuspend`
  - forbid means keeping resumed and forbidding suspend
  - they are called mainly when userspace toggles sysfs `power/control`
- `pm_schedule_suspend` arms a timer to call `rpm_suspend`
- `pm_runtime_barrier` blocks until pending ops are completed or canceled
  - if there is async resume, resume now
  - timer for scheduled suspend and auto suspend is
  - async op (`RPM_ASYNC`) is canceld
  - concurrent op (from another thread) is waited
  - this is mainly called from system suspend or shutdown, to prevent
    in-flight rpm ops
- `pm_runtime_no_callbacks` removes sysfs `power/` (and skips drv rpm callbacks)
  - this is called when the device has no rpm support and relies on its parent
    for rpm (e.g., usb iface relies on its parent usb dev for rpm)
- `pm_runtime_irq_safe` means that the drv will do rpm from atomic context
  - pm core will keep the parent resumed, and will keep spinlock held and irq
    disabled while calling drv callbacks
  - don't use it
- `pm_runtime_set_memalloc_noio` sets `PF_MEMALLOC_NOIO` for allocs from drv
  rpm callbacks
  - this is called by storage drivers to avoid `io -> resume -> alloc -> io`
    deadlock
- `pm_runtime_get_suppliers` and `pm_runtime_put_suppliers` get/put
  all suppliers of the dev's from `DL_FLAG_PM_RUNTIME` devlinks
  - this is called before `really_probe`
- `pm_runtime_new_link` and `pm_runtime_drop_link` track supplier count
  - `dev->power.links_count` is an optimization to avoid taking a lock only to
    learn that `dev->links.suppliers` is empty
- `pm_runtime_release_supplier` decrements supplier usage without suspend
  - when `rpm_resume` is called on a consumer, `rpm_get_suppliers` is called
    on all suppliers
  - this is mainly called from `rpm_put_suppliers`, but is also explicitly
    called when the devlink is to be removed
- `pm_suspend_ignore_children` ignores parent-childen relation
  - that is, don't resume parent when a child is resumed, allow a parent to
    suspend with resumed children, etc.
  - this is used when the parent has multiple functions as "virtual children"
    and it can make better decision for the children by itself
- `pm_runtime_get_noresume` and `pm_runtime_put_noidle` increment/decrement
  usage without resume/suspend
  - `pm_runtime_get_noresume` is useful when
    - dev is known resumed and it is cheaper than `pm_runtime_get`
    - dev may be resumed or suspended, and we just want to block potential rpm
      suspend to avoid conflict with system suspend, etc.
    - dev is resumed on boot and we just want to fix the usage count
    - internal `rpm_resume` cannot call `pm_runtime_get_sync` on the parent
      but has to do increment and resume in two steps
- `pm_runtime_active` and `pm_runtime_suspended` return true if dev is
  resumed/suspended
  - if rpm is disabled, the dev is considered resumed
- `pm_runtime_mark_last_busy` updates `dev->power.last_busy` for auto-suspend
- wrappers of `rpm_resume`, `rpm_suspend`, and `rpm_idle`
  - immediate resume (might sleep)
    - `pm_runtime_resume` calls `rpm_resume(0)`
    - `pm_runtime_get_sync` calls `rpm_resume(0)` after incrementing usage
      - usage incremented on resume failure, useful when caller does not error check
    - `pm_runtime_resume_and_get` calls `rpm_resume(0)` after incrementing usage
      - usage not incremented on resume failure, useful when caller does error check
  - async resume (atomic safe)
    - `pm_request_resume` calls `rpm_resume(RPM_ASYNC)`
    - `pm_runtime_get` calls `rpm_resume(RPM_ASYNC)` after incrementing usage
  - immediate suspend (might sleep)
    - `pm_runtime_suspend` calls `rpm_suspend(0)`
    - `pm_runtime_put_sync_suspend` calls `rpm_suspend(0)` after decrementing usage
  - immediate auto suspend (might sleep, might delay)
    - `pm_runtime_idle` calls `rpm_idle(0)`
    - `pm_runtime_autosuspend` calls `rpm_suspend(RPM_AUTO)`
    - `pm_runtime_put_sync` calls `rpm_idle(0)` after decrementing usage
    - `pm_runtime_put_sync_autosuspend` calls `rpm_suspend(RPM_AUTO)` after decrementing usage
  - async auto suspend (atomic safe, might delay)
    - `pm_request_idle` calls `rpm_idle(RPM_ASYNC)`
    - `pm_request_autosuspend` calls `rpm_suspend(RPM_ASYNC|RPM_AUTO)`
    - `pm_runtime_put` calls `rpm_idle(RPM_ASYNC)` after decrementing usage
    - `pm_runtime_put_autosuspend` calls `rpm_suspend(RPM_ASYNC|RPM_AUTO)` after decrementing usage
- `pm_runtime_set_active` and `pm_runtime_set_suspended` update the
  resumed/suspended status without callbacks
  - when the hw is in a different status than what kernel pm assumes, these
    update the status to match without calling to the callbacks
  - e.g., instead of `pm_runtime_get_sync`, driver probe should do
    `pm_runtime_set_active` and `pm_runtime_get_noresume` instead to fix the
    rpm state if the dev is resumed on boot
- `pm_runtime_use_autosuspend`, `pm_runtime_dont_use_autosuspend`, and
  `pm_runtime_set_autosuspend_delay` config auto-suspend

## Driver Writer

- hw probe
  - runtime pm assumes the device is suspended by default
    - use `pm_runtime_set_active` if the assumption is incorrect
  - `devm_pm_runtime_enable` enables runtime pm
    - it implies both `pm_runtime_dont_use_autosuspend` and
      `pm_runtime_disable` on remove
  - `pm_runtime_use_autosuspend` enables autosuspend
    - `pm_runtime_set_autosuspend_delay` sets a delay
- hw access
  - `pm_runtime_resume_and_get` resumes the dev immediately (for reg access)
    - not `pm_runtime_get_sync` unless too lazy to error check
  - `pm_runtime_get` resumes the dev async (to kick hw)
  - `pm_runtime_put_autosuspend` auto-suspends async
    - not `pm_runtime_put` to avoid overhead of `rpm_idle`
    - it implies `pm_runtime_mark_last_busy`
- hw remove
  - `pm_runtime_put_sync_suspend` suspends the dev immediately
- autosuspend
  - like suspend, except there is a delay that is configurable by the driver
    or the userspace
  - `/sys/devices/.../power/autosuspend_delay_ms`, -1 to disable autosuspend
- e.g., msm uses autosuspend
  - on init, sets the autosuspend delay to `DRM_MSM_INACTIVE_PERIOD` (66ms)
  - on submit, if the submit queue becomes non-empty, `pm_runtime_get`
  - on retire, if the submit queue becomes empty,
    `pm_runtime_put_autosuspend`
