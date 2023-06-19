mkinitcpio
==========

## Usage

- options
  - `-g <output>` writes the initramfs to `<output>`; dry run if not specified
  - `-k <ver>` specifies the kernel version; defult to `uname -r`
  - `-v` be verbose
  - `-L` lists hooks
  - `-H <hook>` shows the help message of `<hook>`
  - `-p <present>` creates an initramfs according to
    `/etc/mkinitcpio.d/<present>.preset`
- `/etc/mkinitcpio.conf` by default enables these hooks in order
  - `base` adds `busybox`, `kmod`, `/init`, and others
  - `udev` adds `systemd-udevd`, `udevadm`, and others
  - `autodetect` scans `/sys` for `modalias`s and invokes `modprobe -R` to map
    them to module names cached in `_autodetect_cache` variable; hooks after
    `autodetect` only consider modules in `_autodetect_cache`
  - `modconf` adds `modprobe` configuration files
  - `kms` adds modules related to drm as well as their dependencies/firmwares
  - `keyboard` adds modules related to input
  - `keymap` adds the keymap if any
  - `consolefont` adds the console font if any
  - `block` adds modules related to block
  - `filesystems` adds modules related to fs
  - `fsck` adds fsck for different filesystems
- `plymouth` hook
  - it adds `plymouth`, the current theme (returned by
    `plymouth-set-default-theme`), and the runscript
