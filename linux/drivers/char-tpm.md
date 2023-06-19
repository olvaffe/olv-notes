Kernel TPM
==========

## TCG (Trusted Computing Group) Specifications

- tpm 2.0
  - hw spec
    - TPM Library Specification
    - <https://trustedcomputinggroup.org/resource/tpm-library-specification/>
  - pc interface spec
    - PTP (Platform TPM Profile) Specification
    - <https://trustedcomputinggroup.org/resource/pc-client-platform-tpm-profile-ptp-specification/>
  - mobile interface spec
    - Mobile CRB (Command Response Buffer) Interface Speficiation
    - <https://trustedcomputinggroup.org/resource/tpm-2-0-mobile-command-response-buffer-interface-specification/>
  - sw spec
    - TSS 2.0
    - <https://trustedcomputinggroup.org/resource/tss-overview-common-structures-specification/>
- tpm 1.2
  - hw spec
    - TPM Main Specification
    - <https://trustedcomputinggroup.org/resource/tpm-main-specification/>
  - pc interface spec
    - TIS (TPM Interface Specification)
    - <https://trustedcomputinggroup.org/resource/pc-client-work-group-pc-client-specific-tpm-interface-specification-tis/>
  - sw spec
    - TSS 1.2
    - <https://trustedcomputinggroup.org/resource/tcg-software-stack-tss-specification/>

## Kernel Configs

- `CONFIG_TCG_TIS` talks TIS (for 1.2) or PTP (for 2.0) over MMIO
  - it registers the platform driver `tis_drv` and the pnp driver
    `tis_pnp_driver`
  - the platform driver matches acpi `MSFT0101` and of `tcg,tpm-tis-mmio`
- `CONFIG_TCG_TIS_SPI` talks TIS or PTP over SPI
  - it registers the spi driver `tpm_tis_spi_driver`
  - the spi driver matches acpi `SMO0768`, of `tcg,tpm_tis-spi`/`google,cr50`,
    or spi `tpm_tis_spi`/`tpm_tis-spi`/`cr50`
  - `CONFIG_TCG_TIS_SPI_CR50` is like a quirk when the tpm is actually cr50
- `CONFIG_TCG_TIS_I2C` talks TIS or PTP over I2C
  - it registers the i2c driver `tpm_tis_i2c_driver`
  - the i2c driver matches i2c `tpm_tis_i2c`
- `CONFIG_TCG_TIS_I2C_CR50` talks TIS or PTP to cr50 over i2c
  - it registers the i2c driver `cr50_i2c_driver`
  - the i2c driver matches acpi `GOOG0005` or of `google,cr50`
- `CONFIG_TCG_CRB` talks CRB 2.0
  - it registers the acpi driver `crb_acpi_driver`
  - the acpi driver matches acpi `MSFT0101`
- my x1 carbon needs `CONFIG_TCG_TIS`
- my chromebooks sets
  - `CONFIG_TCG_TIS`
  - `CONFIG_TCG_TIS_SPI`
  - `CONFIG_TCG_TIS_SPI_CR50`
  - `CONFIG_TCG_TIS_I2C_CR50`
