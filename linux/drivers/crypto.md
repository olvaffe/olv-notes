Kernel Crypto Device
====================

## CCP

- `CONFIG_CRYPTO_DEV_CCP_CRYPTO` registers crypto algorithms to the crypto
  subsystem
  - `ccp_register_rsa_algs` registers `ccp_rsa_defaults` to the crypto
    subsystem
