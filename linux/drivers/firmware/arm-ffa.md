# Kernel ARM FFA

## ARM FFA

- it used to be that
  - EL3 (tfa)
  - sEL1 (optee)
  - sEL0 (ta)
  - normal world uses `smc` to traps to EL3
    - the call follows smccc convention and dispatches to (legacy) trusted os/app
    - tfa jumps to opee in sEL1
- it is becoming
  - sEL2 (hafnium): this works as a secure world hypervisor
    - optee runs in one vm in secure world
  - normal world still uses `smc` to traps to EL3
    - the call follows smccc convention and dispatches to ffa
    - tfa jumps to hafnium in sEL2
- op-tee subsystem supports two transports
  - legacy: smc -> tfa@el3 -> optee@sel1
  - modern: smc -> tfa@el3 -> hafnium@sel2 -> optee@sel1
- FFA is a standard way to talk to TEE (and SPM)
  - hafnium, the sel2 hypervisor, is called an SPM (secure partition manager)
  - optee, an sel1 vm, is called a SP
  - FFA provides ways to enumerate SPs, etc.
- `subsys_initcall(arm_ffa_bus_init)`
  - it registers `arm_ffa` bus to match `ffa_device` and `ffa_driver` by uuids
- `module_init(ffa_init)`
  - it adds `arm-ffa` platdev and registers a plat driver
  - `ffa_probe`
    - `ffa_transport_init`
      - if in EL2, `invoke_ffa_fn` points to `__arm_ffa_fn_smc`
      - if in EL1, `invoke_ffa_fn` points to `__arm_ffa_fn_hvc`
    - `ffa_setup_partitions` discovers ffa devices
      - `ffa_partition_probe` discovers all partitions
      - `ffa_device_register` adds a ffa dev for the partition
- drivers registers `ffa_driver`
  - `module_ffa_driver` or `ffa_register` registers a driver

## Memory Ops

- `FFA_MEM_LEND` lends a memory region to an SP
  - it is unmapped from normal world
- `FFA_MEM_SHARE` shares a memory region with an SP
  - it is kept mapped in normal world
- `FFA_MEM_DONATE` permanently donates a memory region to an SP
- `FFA_MEM_RECLAIM` reclaims a memory region from an SP
  - it fails if the SP hasn't `FFA_MEM_RELINQUISH` the region
