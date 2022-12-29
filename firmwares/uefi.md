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
