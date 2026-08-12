# Kernel ARM PSCI

## ARM PSCI

- standard way to control CPU core states
  - `CPU_ON` powers on a core
  - `CPU_OFF` powers off a core
  - `CPU_SUSPEND` enters low-power state (cpuidle)
  - `SYSTEM_RESET` reboots system
  - `SYSTEM_OFF` powers off system
- conduits
  - smccc is a calling convention
  - smc is an inst that traps to sel3, for bare metal
  - hvc is an inst that traps to el2, for vm guest
