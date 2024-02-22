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
  - OEM sets PK
  - OEM trusts some OS vendors and adds their keys to KEK
  - OS vendors whose keys are in KEK can update db
  - in practice, OEM usually allows users to replace PK
    - with PK replaced, users can update KEK themselves
- linux shim
  - shim is a small bootloader signed by Microsoft using their Third Party key
  - OEM usually adds Microsoft and Microsoft Third Party keys to KEK
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
    KEK, and it only loads executables signed by debian uefi ca (indirectly)
    - this includes official debian grub2, linux, and fwupd

## shim

- <https://github.com/rhboot/shim>
- first stage loader
  - `efi_main` of `shim.c` is the entrypoint
  - `shim_init` picks the compile-time `DEFAULT_LOADER` as the second stage
    loader
    - `DEFAULT_LOADER` is `grubx64.efi` by default
  - `init_grub` calls `start_image` to start the second stage loader
    - if the second stage loader is trusted, the first try succeeds
    - otherwise, `init_grub` calls `start_image` to start the compile-time
      `MOK_MANAGER` and then tries again
      - `MOK_MANAGER` is `mmx86.efi`
- mok (machine owner key) manager
  - it has a ui to enroll keys or hashes
  - hashes
    - use the ui to enroll the hashes of the second stage bootloader and the
      kernel image
  - keys
    - use the ui to enrol the mok key used to sign the second stage bootloader
      and the kernel image
  - to generates a mok key,
    - `openssl req -newkey rsa:4096 -nodes -keyout mok.key -new -x509 -sha256 -days 3650 -subj "/CN=my mok" -out mok.crt`
    - this generates `mok.key` and `mok.crt`
    - `mok.crt` needs to be copied to esp for enrollment
  - to sign a second stage bootloader or a kernel image,
    - `sbsign --key mok.key --cert mok.crt --output <dst> <src>`
  - there is also `mokutil` to enroll keys or hashes from userspace
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
