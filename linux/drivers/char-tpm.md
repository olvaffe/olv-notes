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

## TPM

- each TPM has 3 secret seeds that are burned in
  - the first seed is used for the endorsement hierarchy
    - it is used with a fixed template (algorithm, etc.) to deterministically
      generate the endorsement key (EK)
  - the second seed is used for the platform hierarchy
    - it is used to generate the platform key
    - it is deterministically as long as the template is unchanged
  - the third seed is used for the owner hierarchy
    - it is used to generate the owner key
    - it is deterministically as long as the template is unchanged
  - there is usually an endorsement cert stored in NVRAM
    - the cert is signed by manufacturer and is used to verify the tpm
      itself
- there is another secret seed that is randomly generated on tpm reset
  - this is used for the null hierarchy
  - it is non-deterministically across resets
- chain of trust
  - if there is an endorsement cert, signed by the manufacturer which in turn
    signed by a CA that we can trust, we can trust EK
    - otherwse, we have to trust EK
  - the platform, owner, and null keys are signed by EK so we can trust them
- to create the endorsement/platform/owner/null keys,
  - `tpm2 createprimary -C <hierarchy> -o prim.pub -c prim.ctx`
    - `hierarchy` is `e`/`p`/`o`/`n`
  - in more general tpm terms, this creates an tpm object on tpm
    - each object has a private part and a public part
    - `-o` saves the public part to filesystem
    - `-c` saves the "handle" to filesystem
      - the handle is used to refer to the object
- to create a child object,
  - `tpm2 create -C prim.ctx -u key.pub -r key.priv -c key.ctx`
    - this time, both the public and private parts are saved to filesystem
    - the private part is encrypted by the parent object
  - `echo test | tpm2 create -C prim.ctx -i - -c blob.ctx`
    - this saves a small amount of user data to tpm instead
    - to read back, `tpm2 unseal -c blob.ctx`

## `tpm2-tools`

- `tpm2`
  - <https://github.com/tpm2-software/tpm2-tools>
  - `tpm2 <tool> --help=man` for tool man pages
  - TCTI configuration
    - TCG TSS 2.0 TPM Command Transmission Interface
    - `-T <name>:<config>`
      - default is `-T device:/dev/tpm0`
    - `name` can be
      - `device` talks to the tpm device directly
      - `tabrmd` or `tbrmd` talks to `tpm2-abrmd` daemon
        - <https://github.com/tpm2-software/tpm2-abrmd>
        - Access Broker & Resource Management Daemon
      - `mssim` talks to the sw simulator
      - `none` disables connection to tpm
    - `config` is `name`-specific
  - `getcap` queries tpm caps
    - `-l` to list cap groups
  - `getekcertificate` retrieves endorsement key (EK) certs
    - certs are X.509 certs in DER format
    - `-o` to output to files
    - `openssl x509 -in <file> -text` to decode
  - `getrandom` generates random bytes
  - `readclock` reads the current time
  - `gettime` generates signed current time
  - `hash` hashes data
    - `-g` to specify the algorithm
  - `hmac` performs hmac
    - `-c` to specify the key
  - `import` imports an external key into tpm as a managed key object
  - `load`
  - `loadexternal`
  - `makecredential`
  - `nvdefine`
  - `nvextend`
  - `nvincrement`
  - `nvreadpublic`
  - `nvread`
  - `nvreadlock`
  - `nvundefine`
  - `nvwrite`
  - `nvwritelock`
  - `nvsetbits`
  - `pcrallocate`
  - `pcrevent`
  - `pcrextend`
  - `pcrread`
  - `pcrreset`
  - `quote`
  - `readpublic`
  - `rsadecrypt`
  - `rsaencrypt`
  - `sign`
  - `certifycreation`
  - `nvcertify`
  - `unseal`
  - `verifysignature` verifies a signature
  - `geteccparameters` retrieves the params of an ECC curve
  - `ecephemeral` creates ephemeral key pair for key exchange protocol

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
