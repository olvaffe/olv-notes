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
    - on x86, `native_machine_halt` stops all other cpus and enters `hlt` loop
    - on arm, it stops all others cpus and enters nop loop
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
    - on x86, `native_machine_restart` stops all others cpus and call
      `acpi_reboot`
    - on arm, it stops all others cpus and call `do_kernel_restart`
- `do_kernel_power_off` calls all `SYS_OFF_MODE_POWER_OFF` handlers
  - on x86, `acpi_sleep_init` registers `acpi_power_off`
  - on arm, it is board-dependent
    - `bcm2835_power_off` on raspberry pi
- `do_kernel_restart` calls all `SYS_OFF_MODE_RESTART` handlers and handlers
  registered with `register_restart_handler`
  - x86 does not use this
  - on arm, it is board-dependent
