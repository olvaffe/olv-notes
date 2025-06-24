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
