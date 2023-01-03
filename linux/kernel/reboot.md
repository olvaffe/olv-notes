Reboot
======

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

