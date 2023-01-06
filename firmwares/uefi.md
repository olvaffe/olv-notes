UEFI
====

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
