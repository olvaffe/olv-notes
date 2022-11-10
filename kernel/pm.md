Power Management
================

## APM

- `arch/x86/kernel/apm_32.c`
- there is an apm thread polling the apm hw events.  If suspend or standy
  events are received, the kernel does the suspend/standby.
- there is `/proc/apm_bios` for userspace to check for apm status.

## APM Emulation

- emulate an apm bios on embedded system
- depends on the PMU to provide `apm_get_power_status`
- userspace can control suspend/standby too.

## System level and runtime

- system level manages the power states of the entire system
- runtime manages the power states of each device
- no generic runtime power management available for current kernel
- one cannot echo 'power-off' to a random device

## `reboot` syscall

- does not imply `sync`
- requires superuser
- userspace should stop all services, unmount filesystems, etc. before making
  the syscall
- if in pid namespace, this sends `SIGKILL` to pid 1 (and kernel will send
  `SIGKILL` to all processes in the pid namespace)
- `LINUX_REBOOT_CMD_SW_SUSPEND` is suspend-to-disk
- for others, they first
  - call notifiers registered with `register_reboot_notifier`
  - call `shutdown` on all devices
  - call `shutdown` of syscore ops registered with `register_syscore_ops`
  - stop all other cpus and core hw (arch-specific)
- after the steps above,
  - `LINUX_REBOOT_CMD_HALT`
    - on x86, it executes `hlt` instruction
    - on arm64, it stays in a `while(1);` loop forever
  - `LINUX_REBOOT_CMD_POWER_OFF`
    - on x86, it calls `pm_power_off` which is `acpi_power_off` to cut the
      power
    - on arm64, it calls `pm_power_off` which can be anything
      (e.g., `bcm2835_power_off` on raspberry pi)
  - `LINUX_REBOOT_CMD_RESTART2`
    - on x86, it tries various reboot methods, starting with the one specified
      by `reboot=` or `BOOT_ACPI`.
    - on arm64, it tries various restart handlers registered with
      `register_restart_handler`

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
