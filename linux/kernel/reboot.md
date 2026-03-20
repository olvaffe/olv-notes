Reboot
======

## `reboot` syscall

- userspace should stop all services, unmount filesystems, etc. before making
  the syscall
  - it does not imply `sync`
- it requires `CAP_SYS_BOOT`
- if in pid namespace, `reboot_pid_ns` never returns
  - this sends `SIGKILL` to pid 1 (and kernel will send `SIGKILL` to all
    processes in the pid namespace)
  - `do_exit` does not return
- if `LINUX_REBOOT_CMD_SW_SUSPEND`, `hibernate` suspends to disk
- if `LINUX_REBOOT_CMD_HALT`, `kernel_halt`
  - `kernel_shutdown_prepare`
  - `syscore_shutdown`
  - prints `System halted`
  - `machine_halt` is arch-specific
    - on x86, `native_machine_shutdown` stops other cpus and `stop_this_cpu`
      enters `hlt` loop
    - on arm, `smp_send_stop` stops others cpus and it enters nop loop
- if `LINUX_REBOOT_CMD_POWER_OFF`, `kernel_power_off`
  - `kernel_shutdown_prepare`
    - it calls all handlers registered with `register_reboot_notifier`
    - `device_shutdown` calls `->shutdown` on all devices
  - `do_kernel_power_off_prepare`
    - it calls all `SYS_OFF_MODE_POWER_OFF_PREPARE` handlers
  - `syscore_shutdown`
    - it calls all handlers registered with `register_syscore`
  - prints `Power down`
  - `machine_power_off` is arch-specific
    - on x86 and arm, they both stop all others cpus and call
      `do_kernel_power_off`
- if `LINUX_REBOOT_CMD_RESTART2`, `kernel_restart`
  - `kernel_restart_prepare` is similar to `kernel_shutdown_prepare`
  - `do_kernel_restart_prepare`
    - it calls all `SYS_OFF_MODE_RESTART_PREPARE` handlers
  - `syscore_shutdown`
  - prints `Restarting system ...`
  - `machine_restart` is arch-specific
    - on x86, `native_machine_shutdown` stops others cpus and
      `native_machine_emergency_restart` calls `acpi_reboot`
    - on arm, `smp_send_stop` stops others cpus and it calls
      `do_kernel_restart`
- `do_kernel_power_off` calls all `SYS_OFF_MODE_POWER_OFF` handlers
  - on x86, `acpi_power_off` registered by `acpi_sleep_init` enters S5
  - on arm, often `psci_sys_poweroff` calls `PSCI_0_2_FN_SYSTEM_OFF`
- `do_kernel_restart` calls all `SYS_OFF_MODE_RESTART` handlers and handlers
  registered with `register_restart_handler`
  - x86 does not use this
  - on arm, often `psci_sys_reset` calls `PSCI_0_2_FN_SYSTEM_RESET`
