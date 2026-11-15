# Kernel ARM SMCCC

## Overview

- `smc` instruction traps to EL3
  - in a vm, `hvc` traps to EL2 instead
- smccc defines a calling convention (abi) to call secure world services
  - `x0` holds `ARM_SMCCC_CALL_VAL` and returns status
    - `ARM_SMCCC_TYPE_SHIFT`
      - `ARM_SMCCC_STD_CALL`
      - `ARM_SMCCC_FAST_CALL` is atomic with irq disabled
    - `ARM_SMCCC_CALL_CONV_SHIFT`
      - `ARM_SMCCC_SMC_32`
      - `ARM_SMCCC_SMC_64`
    - `ARM_SMCCC_OWNER_SHIFT`
      - `ARM_SMCCC_OWNER_ARCH` is defined by cpu arch
      - `ARM_SMCCC_OWNER_CPU` is defined by cpu impl for erratas
      - `ARM_SMCCC_OWNER_SIP` is defined by soc (e.g., qcom scm)
      - `ARM_SMCCC_OWNER_OEM` is defined by board oem
      - `ARM_SMCCC_OWNER_STANDARD` is defined by standard services (psci, ffa, etc.)
      - `ARM_SMCCC_OWNER_STANDARD_HYP`
      - `ARM_SMCCC_OWNER_VENDOR_HYP`
      - `ARM_SMCCC_OWNER_TRUSTED_APP..` is defined by TAs (legacy)
      - `ARM_SMCCC_OWNER_TRUSTED_OS..` is defined by TEEs (legacy)
  - `x1..x3` are arg ins and outs
  - `x4..x17` are arg ins and outs (since 1.2)
  - `x18..x30` and `sp` are callee-saved
- EL3 (tfa) dispatches based on type and owner
  - `ARM_SMCCC_OWNER_{ARCH,CPU}` dispatch to standard handlers in EL3
  - `ARM_SMCCC_OWNER_{SID,OEM}` dispatch to soc/board hooks in EL3
  - `ARM_SMCCC_OWNER_STANDARD` dipatches to standard service handler in EL3
    which performs second-level dispatch
    - PSCI is handled in EL3
    - SDEI is handled in EL3
    - TRNG is handled in EL3
    - SPMD (ffa) dispatches to sEL2 (hafnium) or sEL1 (optee)
    - RMMD dispatches to rEL2 (tf-rmm)
  - `ARM_SMCCC_OWNER_TRUSTED_{APP,OS}` dispatch to legacy soc/board hooks in EL3
    - the hooks are called `SPD` and they switch to sEL1
    - they are being replaced by standard ffa
