UEFI
====

## TianoCore EDK II

- the repo is at <https://github.com/tianocore/edk2>
  - a stable release is tagged every 3 month
- history
  - in 2004, intel released their reference EFI impl
    - the name Tiano was presented in the code
    - used as the basis for community-run project, EDK, EFI Development Kit
  - in 2006, intel released edk2
    - the code was also referred to as Tiano R9
  - <https://github.com/tianocore/tianocore.github.io/wiki/UDK>
    - historically, intel took a snapshot of edk2, validated the code, and
      released it as UDK
    - UDK{2008,2010,2014,2015,2017,2018}

## Variables

- variables are namespaced by guids
  - `/sys/firmware/efi/efivars/<name>-<guid>`
- `EFI_GLOBAL_VARIABLE` (8be4df61-93ca-11d2-aa0d-00e098032b8c)
  - `Boot####` are boot entries containing descriptive names and paths to
    bootloaders
  - `BootCurrent` is a u16 for current `Boot####`
  - `BootOrder` is a list of u16 for ordered `Boot####`
  - `KEK` is an array of `EFI_SIGNATURE_LIST` for KEK
    - a `EFI_SIGNATURE_LIST` is a cert
    - because the variable has
      `EFI_VARIABLE_TIME_BASED_AUTHENTICATED_WRITE_ACCESS` flag, writing to
      the variable requires preceding `EFI_SIGNATURE_LIST` array by
      `EFI_VARIABLE_AUTHENTICATION_2`
  - `Key####` is hotkey for `Boot####`
  - `OsIndications` is for os to configure the next boot
    - `EFI_OS_INDICATIONS_BOOT_TO_FW_UI` requests booting to bios
  - `PK` is an array of `EFI_SIGNATURE_LIST` for PK
  - `PlatformLang` is the lang code such as `en-US`
  - `SecureBoot` is 1 if enabled
  - `SetupMode` is 1 if `PK` is missing
  - `SignatureSupport` is an array of guids for supported certs (x509, etc.)
  - `Timeout` is the timeout before booting `Boot####`
  - `VendorKeys` is 1 if factory keys are used
- `EFI_IMAGE_SECURITY_DATABASE_GUID` (d719b2cb-3d3a-4596-a3bc-dad00e67656f)
  - `db` is an array of `EFI_SIGNATURE_LIST` for db
  - `dbx` is an array of `EFI_SIGNATURE_LIST` for dbx
- 4a67b082-0a4c-41cf-b6c7-440b29bb8c4f
  - <https://systemd.io/BOOT_LOADER_INTERFACE/>
- `SHIM_LOCK_GUID` (605dab50-e046-4300-abb6-3dd810dd8b23)
  - <https://github.com/rhboot/shim>

## Secure Boot

- there should be a security module acting as root-of-trust to verify the uefi
  firmware before the uefi firmware is executed
- the uefi firmware implements uefi secure boot
  - chain of trust
    - platform key, PK
    - key exchange key, KEK
    - signature database, db
  - machine owner generates PK
    - PK is x509 cert
    - PK is only used to sign KEK updates
      - it is used rarely and can be stored offline for highest security
    - there is only one PK
  - machine owner updates KEKs
    - KEK is x509 cert
    - KEK is only used to sign db updates
      - it is used rarely and can be stored offline for highest security
    - there can be multiple KEKs, one for each trusted OS vendor
  - OS vendors updates db keys
    - db key is x509 cert
    - db key is used to sign executables
      - it is used frequently and is stored online for convenience
    - there can be multiple db keys
      - each trusted OS vendor updates db with its db key
        - used to sign its bootloaders and/or kernels
      - if an OS allows uefi firmware update signed by machine OEM, it updates
        db with machine OEM db key
      - if an OS allows uefi executables signed by MS, it updates db with MS
        db keys
        - there are 5 MS db keys according to sbctl
        - this is often needed for dgpu, whose option rom is an executable
        - this is a secure concern for some though
- <https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-secure-boot-key-creation-and-management-guidance?view=windows-11>
  - PK: OEM
  - KEK
    - OEM
    - Microsoft Corporation KEK 2K CA 2023
      - previously Microsoft Corporation KEK CA 2011
  - db
    - OEM, for firware update
    - Windows UEFI CA 2023, to boot windows
      - previouly Microsoft Windows Production PCA 2011
    - Microsoft UEFI CA 2023, for third-party executables
      - previously Microsoft Corporation UEFI CA 2011
    - Microsoft Option ROM UEFI CA 2023, for third-party option roms
- bios settings
  - secure boot can be enabled/disabled
  - PK/KEK/db can be reset to factory default
    - OEM PK
    - OEM and MS KEKs
    - OEM and MS db keys
  - PK/KEK/db can be cleared
    - this puts the machine in setup mode, where a new PK key can be set
- personal machine
  - personal PK
  - personal KEK
  - personal, OEM, and MS db keys

## sbctl

- setup
  - `sbctl export-enrolled-keys --dir backup --disable-landlock` backs up
    existing keys
  - `sbctl create-keys` creates new keys under `/var/lib/sbctl`
  - `sbctl enroll-keys` enrolls new keys
    - `-f` to also enroll fw-defined keys
      - this uses `{PK,KEK,db}Default` variables and might not work
    - `-m` to also enroll microsoft keys
    - `-c` to also enroll custom keys from `/var/lib/sbctl/keys/custom`
      - `sbctl export-enrolled-keys --dir /var/lib/sbctl/keys/custom --disable-landlock`,
        and rename `/var/lib/sbctl/keys/custom/DB` to `/var/lib/sbctl/keys/custom/db`
  - `sbctl sign <file>` signs a file in-place
    - `-o <out>` to output to a separate file instead
    - `-s` to remember the file in `/var/lib/sbctl/files.json`
  - `sbctl sign-all` signs all remembered files
- `/usr/share/libalpm/hooks/zz-sbctl.hook`
  - it invokes `sbctl sign-all -g` when there are package changes to
    - `boot/*` (e.g., intel-ucode)
    - `usr/lib/modules/*/vmlinuz` (e.g., linux)
    - `usr/lib/**/efi/*.efi*` (e.g., systemd)
- `/usr/lib/initcpio/post/sbctl`
  - it invokes `sbctl sign` on vmlinuz/uki created by `mkinitcpio`
- `/usr/lib/kernel/install.d/91-sbctl.install`
  - it invokes `sbctl sign` on vmlinuz/uki created by `kernel-install`

## ukify

- setup
  - `cp /usr/lib/kernel/uki.conf /etc/kernel` and edit
  - `ukify genkey --config /etc/kernel/uki.conf` to generate privkey and cert
  - `/usr/lib/systemd/systemd-sbsign sign ...` to sign the bootloader
  - `bootctl install --secure-boot-auto-enroll yes ..` to install signed
    bootloader and stage keys for enrollment
    - THE KEYS MAY BRICK THE MACHINE ESPECIALLY ON THINKPAD ONCE ENROLLED
    - `man loader.conf` has examples to generate working keys
  - `echo "secure-boot-enroll manual" >> /boot/loader/loader.conf` to enroll
    keys on next boot

## shim

- <https://github.com/rhboot/shim>
  - shim is a small bootloader signed by Microsoft using their Third Party key
  - OEM usually adds Microsoft and Microsoft Third Party keys to db
  - together, shim can be verified and booted
  - shim then implements its own chain of trust
    - it contains a distro key that is used to verify that the real bootloader
      is signed by the distro
- <https://github.com/rhboot/shim-review> is how a distro gets its build of
  shim plus distro certificate signed by microsoft
  - e.g., <https://github.com/rhboot/shim-review/issues/315> is for shim 15.7
    on debian bookworm
  - the unsigned shim embeds a cert for debian uefi ca
  - the signed shim is signed by MS 3rd party key
  - IOW, the signed shim is bootable on any machine with ms 3rd party key in
    db, and it only loads executables signed by debian uefi ca (indirectly)
    - this includes official debian grub2, linux, and fwupd
- first stage loader
  - `efi_main` of `shim.c` is the entrypoint
  - `shim_init` picks the compile-time `DEFAULT_LOADER` as the second stage
    loader
    - `DEFAULT_LOADER` is `grubx64.efi` by default
  - `init_grub` calls `start_image` to start the second stage loader
    - if the second stage loader is trusted, the first try succeeds
    - otherwise, `init_grub` calls `start_image` to start the compile-time
      `MOK_MANAGER` and then tries again
      - `MOK_MANAGER` is `mmx64.efi`
- mok (machine owner key) manager
  - it is an uefi app with ui to enroll keys or hashes
    - to disallow enrollment, do not copy `mmx64.efi` to esp
  - enroll custom keys
    - need to create custom key once
    - need up sign bootloader and kernel image on every update
  - enroll hashes
    - need to re-enroll bootloader and kernel image on every update
- there is also `mokutil` to stage keys or hashes for enrollment from userspace
- image signing
  - to generates a mok key,
    - `openssl req -newkey rsa:2048 -noenc -keyout mok.key -new -x509 -sha256 -days 3650 -subj "/CN=my mok" -out mok.crt`
    - this generates `mok.key` and `mok.crt`
    - `mok.crt` needs to be copied to esp for enrollment
  - to sign a second stage bootloader or a kernel image,
    - `/usr/lib/systemd/systemd-sbsign sign --private-key mok.key --certificate mok.crt --output <dst> <src>`
- in arch linux,
  - the official iso is unsigned
  - the official `shim` package is unsigned
  - only aur `shim-signed` package is signed

## GPT disk

- ESP
  - there should be a bootloader for each OS
    - though some bootloaders can load multiple OS kernels
  - Bootloaders should be in `EFI/` subdirectory
  - Windows bootloader is at `EFI/Microsoft/bootmgfw.efi`
  - EFI provides its own bootloader, which is a boot manager that allows one to
    choose another bootloader
    - rEFIt is a better choice for a boot manager
  - Copy your bootloader to `EFI/Boot/bootx64.efi` to make it the default
  - Windows 7 insists ESP to be FAT32.  So use it.
  - `efibootmgr` is a userspace tool to manipulate `EFI/`

## EDK2 PXE

- `EfiPxeBcDiscover` performs the pxe discovery sequence
  - `PxeBcDhcp4Discover` builds the dhcpv4 discovery packet
    - it asks for a bunch of dhcp options
  - `PxeBcParseDhcp4Packet` parses the dhcpv4 offer packet
    - `mInterestedDhcp4Tags` lists the options to parse
    - if there is no dhcp message type option (53), this is a bootp offer
      - the boot file is given by boot file option (67)
    - else if there is the vendor class id option (60) and it is
      `PXEClient`, this is a pxe offer
      - if there is the vendor specific option (43), it is further parsed to
        distinguish between pxe 1.0 and binl (microsoft?)
    - else, this is a dhcpv4-only offer
