# Kernel ARM FFA

## ARM FFA

- standard way to talk to SPM / TEE
- op-tee subsystem supports two transports
  - legacy: smc -> tfa@sel3 -> optee@sel1
  - modern: ffa -> spmc@sel2 -> optee@sel1
