Kernel Crypto Device
====================

## AMD CCP

- AMD Secure Processor is formerly known as Platform Security Processor, or
  PSP
- PSP has 5 major components
  - BootROM, or on-chip bootloader
  - Initial Program Loader (IPL), or off-chip bootloader
  - AMD Generic Encapsulated Software Architecture (AGESA) bootloader
  - secure OS
  - Cryptographic Co-Processor (CCP)
- `CONFIG_CRYPTO_DEV_CCP_DD` enabes PSP support
  - `sp_mod_init` calls `sp_pci_init` to register a pci driver,
    `sp_pci_driver`
  - `sp_init` calls `ccp_dev_init` only when `CONFIG_CRYPTO_DEV_SP_CCP`
- `CONFIG_CRYPTO_DEV_SP_CCP` enables CCP support
  - `sp_init` calls `ccp_dev_init` to init ccp
  - for `ccp3_actions`, the init function is `ccp_init`
    - `ccp_register_rng` calls `hwrng_register` to register a hwrng
    - `ccp_dmaengine_register` calls `dma_async_device_register` to register a
      dma engine
- `CRYPTO_DEV_CCP_CRYPTO`
  - `ccp_crypto_init`
  - `ccp_register_aes_algs` calls `crypto_register_skcipher` to register
    various aes algs
    - e.g., when `name` is `ecb(aes)`, `driver_name` is `ecb-aes-ccp`

## CCP

- `CONFIG_CRYPTO_DEV_CCP_CRYPTO` registers crypto algorithms to the crypto
  subsystem
  - `ccp_register_rsa_algs` registers `ccp_rsa_defaults` to the crypto
    subsystem
