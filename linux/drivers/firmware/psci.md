# Kernel ARM PSCI

## ARM PSCI

- PSCI is directly implemented by el3 (e.g., tfa)
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
- take `PSCI_1_0_FN64_SYSTEM_SUSPEND` encoding for example
  - `PSCI_0_2_FN_BASE` is `0x84000000`, same as
    - `ARM_SMCCC_FAST_CALL << ARM_SMCCC_TYPE_SHIFT`
    - `ARM_SMCCC_OWNER_STANDARD << ARM_SMCCC_OWNER_SHIFT`
  - `PSCI_0_2_64BIT` is `0x40000000`, same as
    - `ARM_SMCCC_SMC_64 << ARM_SMCCC_CALL_CONV_SHIFT`
  - a psci call follows smccc convention
