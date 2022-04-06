Debian
======

## `debootstrap`

- `debootstrap` does 4 things
  - download deb packages
  - unpack them into the target directory
  - chroot into the target directory
    - this requires root
  - run the installation and configuration scripts from the packages
    - this forbids cross-compiling
- `sudo debootstrap stable stable-chroot`
  - or replace `sudo` by `fakechroot fakeroot`

## cross-`debootstrap`

- `debootstrap --arch arm64 --foreign stable stable-chroot`
  - `--foreign` skips the last two steps
- `apt install qemu-user-static`
- `cp /usr/bin/qemu-aarch64-static stable-chroot`
- `sudo chroot stable-chroot /qemu-aarch64-static /bin/bash -i`
- `/debootstrap/debootstrap --second-stage`

## cross-compile

- `apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu`
- in chroot, install whatever dependent -dev packages
