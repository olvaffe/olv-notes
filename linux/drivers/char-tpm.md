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

- TPM chip hw components
  - secure processor
    - for dTPMs (discrete TPMs), it is typically an ARM core
    - for fTPMs (firmware TPMs), it is typically a secure coprocessor (amd) or
      TEE of the main processor (intel)
  - NVRAM for persistent storage
    - internal state and config
    - hierarchy seeds (random values) for endorcement, platform, and owner
      hierarchies
    - user data (NV indices)
  - volatile RAM for temporary stoage
  - crypto engines for accelerated crypto ops
  - io to communicate with the host
- `tpm2 getcap handles-transient` lists object handles in volatile memory
- `tpm2 getcap handles-persistent` lists object handles in nvmem
  - 0x810000XX: storage primary keys
  - 0x810100XX: endorsement primary keys
  - 0x818000XX: platform keys
- `tpm2 getcap handles-permanent` lists object handles in nvmem ro region
  - 0x40000001: `TPM_RH_OWNER`, primary hierarchy
  - 0x40000007: `TPM_RH_NULL`, null hierarchy
  - 0x40000009: `TPM_RS_PW`, password
  - 0x4000000A: `TPM_RH_LOCKOUT`
  - 0x4000000B: `TPM_RH_ENDORSEMENT`, endorsement hierarchy
  - 0x4000000C: `TPM_RH_PLATFORM`, platform hierarchy
  - 0x4000000D: `TPM_RH_PLATFORM_NV`
- `tpm2 getcap handles-pcr` lists PCR handles
- `tpm2 getcap handles-nv-index` lists NV index handles
  - 0x01C00002: RSA 2048 EK Certificate
    - this is x509 cert of RSA EK, signed by root CA, as root of trust
  - 0x01C0000A: ECC NIST P256 EK Certificate
    - this is x509 cert of ECC EK, signed by root CA, as root of trust
  - 0x01C1XXXX: defined by component oem
  - 0x01C2XXXX: defined by tpm oem
  - 0x01C3XXXX: defined by platform oem
- each TPM has 3 secret seeds that are burned in nvmem ro region
  - the first seed is used for the endorsement hierarchy
    - it is used with a fixed template (algorithm, etc.) to deterministically
      generate the endorsement key (EK)
  - the second seed is used for the platform hierarchy
    - it is used to generate the platform key
    - it is deterministically as long as the template is unchanged
  - the third seed is used for the owner hierarchy
    - it is used to generate the owner key
    - it is deterministically as long as the template is unchanged
  - seeds are cheaper to store than keys
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
  - by trusting EK, we trust the TPM chip as a whole
- to create the endorsement/platform/owner/null keys,
  - `tpm2 createprimary -C <hierarchy> -o prim.pub -c prim.ctx`
    - `hierarchy` is `e`/`p`/`o`/`n`
  - in more general tpm terms, this creates a tpm object on tpm
    - the object is a key which has a private part and a public part
    - `-o` saves the public part to filesystem
    - `-c` saves the "handle" to filesystem
      - the handle is used to refer to the private part
- to create a child object,
  - `tpm2 create -C prim.ctx -u key.pub -r key.priv -c key.ctx` creates a key
    - this time, both the public and private parts are saved to filesystem
    - the private part is encrypted by the parent object
    - the context file appears to be a serialization of the loaded key
      - `tpm2 load -C prim.ctx -u key.pub -r key.priv -c key.ctx` loads the
        key and serializes it to the context file again
    - to sign a message with the key,
      - `tpm2 sign -c key.ctx -o msg.sig msg.dat` signs the message
      - `tpm2 verifysignature -c key.ctx -s msg.sig -m msg.dat` verifies the
        signature
  - `echo test | tpm2 create -C prim.ctx -i - -c blob.ctx` creates a sealing object
    - this saves a small amount of user data to tpm
    - to read back, `tpm2 unseal -c blob.ctx`
- PCRs
  - Platform Configuration Registers
  - there are 24 PCRs
    - they are registers that can be read or extended
      - extension means `pcr-x = hash(pcr-x + new-data)`
    - there are usually 2 banks, for sha1 and sha256
    - `tpm2 pcrread` dumps the current values
  - usage
    - we can seal the key to disk encryption to tpm
    - tpm would unseal it when PCR values match pre-calculated values
    - this makes sure the disk is unlocked only when all code and data used
      before unlock are not tampered
  - <https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification/>
    - pcr 0-7 is reserved for firmware (uefi)
    - pcr 8-15 is reserved for os
    - pcr 16 is for debug
    - pcr 23 is for app support
  - pcr-0 is for SRTM, BIOS, Host Platform Extensions, Embedded Option ROMs
    and PI Drivers
    - e.g., the firmware measures itself to pcr-0; the value may change after
      firmware update
  - pcr-1 is for Host Platform Configuration
    - e.g., the firmware measures its config to pcr-1; the value may change
      after config change
  - pcr-2 is for 2 UEFI driver and application Code
    - e.g., the firwware measures uefi drivers and apps to pcr-2
  - pcr-3 is for 3 UEFI driver and application Configuration and Data
    - e.g., the firwware measures uefi driver and app configs to pcr-3
  - pcr-4 is for UEFI Boot Manager Code (usually the MBR) and Boot Attempts
    - e.g., the firwware measures the bootloader to pcr-4; the vlaue may
      change after bootloader update
  - pcr-5 is for Boot Manager Code Configuration and Data (for use by the Boot
    Manager Code) and GPT/Partition Table 
    - e.g., the firwware measures the bootloader config and the partition
      table to pcr-5; the vlaue may change after partition table change
  - pcr-6 is for Host Platform Manufacturer Specific
    - it is reserved for motherboard manufacturer
  - pcr-7 is for Secure Boot Policy
    - e.g., the firwware measures secure boot related variables (SecureBoot,
      PK, KEK, DB, DBX, etc.) to pcr-7
- <https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/>
  - pcr-8 is used by grub for cmdline
  - pcr-9 is used by grub for all files read
  - pcr-10 is used by kernel or ima
  - pcr-11 is used by systemd-stub for uki and by systemd-pcrphase
  - pcr-12
  - pcr-13
  - pcr-14
  - pcr-15
- policies
  - `TPM2_PolicyPCR` creates a policy based on fixed PCR values, to unseal
    secret only when PCRs have the fixed values
  - `TPM2_PolicyAuthorize` creates a policy based on a public key
    - what it does is that, if another policy is signed by the corresponding
      private key, use the policy to unreal secret
    - it enables unsealing based on dynamic PCR values when used with
      `TPM2_PolicyPCR`

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
  - `activatecredential`
  - `certify`
  - `certifycreation`
  - `certifyX509certutil`
  - `changeauth`
  - `changeeps`
  - `changepps`
  - `checkquote`
  - `clear`
  - `clearcontrol`
  - `clockrateadjust`
  - `commit`
  - `create`
  - `createak`
  - `createek`
  - `createpolicy`
  - `createprimary`
  - `dictionarylockout`
  - `duplicate`
  - `ecdhkeygen`
  - `ecdhzgen`
  - `ecephemeral` creates ephemeral key pair for key exchange protocol
  - `encodeobject`
  - `encryptdecrypt`
  - `eventlog`
  - `evictcontrol`
  - `flushcontext`
  - `getcap` queries tpm caps
    - `-l` to list cap groups
  - `getcommandauditdigest`
  - `geteccparameters` retrieves the params of an ECC curve
  - `getekcertificate` retrieves endorsement key (EK) certs
    - certs are X.509 certs in DER format
    - `-o` to output to files
    - `openssl x509 -in <file> -text` to decode
  - `getpolicydigest`
  - `getrandom` generates random bytes
  - `getsessionauditdigest`
  - `gettestresult`
  - `gettime` generates signed current time
  - `hash` hashes data
    - `-g` to specify the algorithm
  - `hierarchycontrol`
  - `hmac` performs hmac
    - `-c` to specify the key
  - `import` imports an external key into tpm as a managed key object
  - `incrementalselftest`
  - `load`
  - `loadexternal`
  - `makecredential`
  - `nvcertify`
  - `nvdefine`
  - `nvextend`
  - `nvincrement`
  - `nvread`
  - `nvreadlock`
  - `nvreadpublic`
  - `nvsetbits`
  - `nvundefine`
  - `nvwrite`
  - `nvwritelock`
  - `pcrallocate`
  - `pcrevent`
  - `pcrextend`
  - `pcrread`
  - `pcrreset`
  - `policyauthorize`
  - `policyauthorizenv`
  - `policyauthvalue`
  - `policycommandcode`
  - `policycountertimer`
  - `policycphash`
  - `policyduplicationselect`
  - `policylocality`
  - `policynamehash`
  - `policynv`
  - `policynvwritten`
  - `policyor`
  - `policypassword`
  - `policypcr`
  - `policyrestart`
  - `policysecret`
  - `policysigned`
  - `policytemplate`
  - `policyticket`
  - `print`
  - `quote`
  - `rc_decode`
  - `readclock` reads the current time
  - `readpublic`
  - `rsadecrypt`
  - `rsaencrypt`
  - `selftest`
  - `send`
  - `sessionconfig`
  - `setclock`
  - `setcommandauditstatus`
  - `setprimarypolicy`
  - `shutdown`
  - `sign`
  - `startauthsession`
  - `startup`
  - `stirrandom`
  - `testparms`
  - `tr_encode`
  - `unseal`
  - `verifysignature` verifies a signature
  - `zgen2phase`

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
