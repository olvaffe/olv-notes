Power Management
================

## APM

* `arch/x86/kernel/apm_32.c`
* there is an apm thread polling the apm hw events.  If suspend or standy
  events are received, the kernel does the suspend/standby.
* there is `/proc/apm_bios` for userspace to check for apm status.

## APM Emulation

* emulate an apm bios on embedded system
* depends on the PMU to provide `apm_get_power_status`
* userspace can control suspend/standby too.

## System level and runtime

* system level manages the power states of the entire system
* runtime manages the power states of each device
* no generic runtime power management available for current kernel
* one cannot echo 'power-off' to a random device
