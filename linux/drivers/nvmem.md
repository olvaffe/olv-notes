Kernel NVMEM
============

## Core

- this subsys is mainly used for otp or efuse
- `devm_nvmem_register` registers an `nvmem_device`
  - `nvmem_config` describes the nvmem
    - `type` is `NVMEM_TYPE_OTP` for otp/efuse
    - `reg_read` is the read op

## Consumer

- consumer dt references the nvmem cells, such as
  - `nvmem-cells = <&dp_calibration>;`
  - `nvmem-cell-names = "dp_calibration_data";`
- consumer driver calls `nvmem_cell_get` to get the cell and `nvmem_cell_read`
  to read the value
