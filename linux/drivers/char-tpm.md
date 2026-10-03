# Kernel TPM

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
- each TPM has 4 secret seeds
  - EPS, endorsement primary seed
    - it is controlled by tpm manufacturer
      - tpm manufacturer runs `TPM2_ChangeEPS` once to generate EPS in the
        chip factory
      - it never changes on consumer deivces
  - PPS, platform primary seed
    - it is controlled by motherboard manufacturer
      - mb manufacturer runs `TPM2_ChangePPS` once to generate PPS in the mb
        factory
      - it almost never changes unless mb goes through RMA or the like
  - SPS, storage primary seed, aka owner primary seed
    - it is controlled by device owner
      - uefi runs `TPM_Clear` to generate SPS
      - bios provides a "clear tpm" function to re-generate
  - null seed
    - it is re-generated every reboot
- a seed generates a transient key deterministically
  - a seed is much smaller than a key to store in nvmem
  - EPS generates EK, endorcement key
    - there are typically RSA EK and ECC EK, both generated deterministically
  - PPS generates PPK, platform primary key
  - SPS generates SRK, storage root key
- chain of trust
  - in the chip factory, tpm manufacturer
    - generates EPS, which never changes
    - generates EK, which is transient
    - generates a cert for EK, which is stored in nvmem ro and never changes
  - by trusting cert, we trust EK and the tpm chip as a whole
    - that is, the tpm chip is authentic, not forged
- to create the endorsement/platform/owner/null keys,
  - `tpm2 createprimary -C <hierarchy> -o prim.pub -c prim.ctx`
    - `hierarchy` is `e`/`p`/`o`/`n`
  - it generates the key pair deterministically in tpm ram, saves the pubkey
    to `prim.pub`, saves the encrypted context (including the privkey) to
    `prim.ctx`, and flushes the key pair from tpm ram
    - tpm ram is very small
    - the context must be loaded into tpm ram everytime before any operation
- to create a child object,
  - `tpm2 create -C prim.ctx -u key.pub -r key.priv -c key.ctx` creates a key
  - it generates the key pair with trng in tpm ram, saves the pubkey to
    `key.pub`, saves the privkey encrypted by `prim.ctx` to `key.priv`, saves
    the encrypted context (including the privkey) to `key.ctx`, and flushes
    the key pair from tpm ram
    - primary key pairs are generated deterministically; the privkeys are not
      saved
    - non-primary key pairs are generated using trng; the privkeys are saved
      - they are encrypted by the primary keys
    - the context must be loaded into tpm ram everytime before any operation
- to use a child object,
  - to sign a message with the key,
    - `tpm2 sign -c key.ctx -o msg.sig msg.dat` signs the message
    - `tpm2 verifysignature -c key.ctx -s msg.sig -m msg.dat` verifies the
      signature
- to seal/unseal user data,
  - `echo test | tpm2 create -C prim.ctx -i - -c blob.ctx` creates a sealing object
    - this saves a small amount of user data to tpm
  - to read back, `tpm2 unseal -c blob.ctx`
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

## Platform Configuration Registers (PCRs)

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
  - definitions
    - pcr 0-7 is reserved for firmware (uefi)
    - pcr 8-15 is reserved for os
    - pcr 16 is for debug
    - pcr 23 is for app support
  - modern recommentations
    - do not use pcr 0-6 to allow bios/bootloader/kernel update or config
      change
    - prefer pcr 11 over pcr 7
  - `TPM2_PCR_PLATFORM_CODE` (0) measures uefi code, etc.
    - it changes after bios update
  - `TPM2_PCR_PLATFORM_CONFIG` (1) measures uefi config, etc.
    - it changes after bios config change (e.g., boot order)
  - `TPM2_PCR_EXTERNAL_CODE` (2) measures option roms, etc.
    - it changes after plugging a pcie gpu, nic, scsi hba, etc.
  - `TPM2_PCR_EXTERNAL_CONFIG` (3) measures option rom configs, etc.
    - it changes after nic pxe config or scsi hba raid config change, etc.
  - `TPM2_PCR_BOOT_LOADER_CODE` (4) measures pe binaries loaded by `LoadImage`
    - it changes after bootloader, kernel, initrd updates
  - `TPM2_PCR_BOOT_LOADER_CONFIG` (5) measures disk/bootloader/kernel/initrd
    - it changes after partition table or bootloader config change, etc.
  - `TPM2_PCR_HOST_PLATFORM` (6) is reserved for motherboard manufacturer
    - it is rarely used nor changes
  - `TPM2_PCR_SECURE_BOOT_POLICY` (7) measures Secure Boot Policy
    - it changes with secure boot related uefi variables: SecureBoot, PK, KEK,
      DB, DBX, etc.
  - `TPM2_PCR_DEBUG` (16) is reserved for debug/test
  - `TPM2_PCR_APPLICATION_SUPPORT` (23) is reserved for userspace apps
- <https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/>
  - pcr-8 measures grub config
  - `TPM2_PCR_KERNEL_INITRD` (9) measures initrd/cmdline and nvpcr
    - kernel itself measures initrd/cmdline
    - systemd measures nvpcr
      - systemd uses nvram as pseudo PCRs
      - the measurement ensures no tampering
  - `TPM2_PCR_IMA` (10) measures userspace binaries and configs
    - kernel ima subsystem measures userspace at runtime
  - `TPM2_PCR_KERNEL_BOOT` (11) measures uki and boot pharses
    - `systemd-stub` measures uki pe sections in canonical order
    - `systemd-pcrextend` measures boot phases
  - `TPM2_PCR_KERNEL_CONFIG` (12) measures dynamic configs of uki
    - it changes with cmdline override, etc.
  - `TPM2_PCR_SYSEXTS` (13) measures sysext disk images
  - `TPM2_PCR_SHIM_POLICY` (14) is reserved for shim
    - shim measures mok and sbat
  - `TPM2_PCR_SYSTEM_IDENTITY` (15) measures luks key/uuid, rootfs partition,
    and machine id
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
