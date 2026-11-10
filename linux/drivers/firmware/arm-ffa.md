# Kernel ARM FFA

## ARM FFA

- standard way to talk to SPM / TEE
- op-tee subsystem supports two transports
  - legacy: smc -> tfa@sel3 -> optee@sel1
  - modern: ffa -> spmc@sel2 -> optee@sel1
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
