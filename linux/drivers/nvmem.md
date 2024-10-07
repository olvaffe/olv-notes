Kernel NVMEM
============

## Core

- this subsys is mainly used for otp or efuse
- `devm_nvmem_register` registers an `nvmem_device`
  - `nvmem_config` describes the nvmem
    - `type` is `NVMEM_TYPE_OTP` for otp/efuse
    - `reg_read` is the read op
