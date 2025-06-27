systemd-cryptsetup
==================

## Setup

- `systemd-cryptenroll /dev/sda2 --tpm2-device=auto`
  - this adds a new key slot where the volume key is encrypted by tpm, and
    adds metadata needed to use tpm to decrypt the volume key
  - `--tpm2-pcrs=7` is the default and it adds the current value of pcr 7 to
    the metadata
  - on boot, when tpm is requrested to decrypt the volume key, it proceeds
    only if pcr 7 has the same value as in the metadata
- on boot,
  - `systemd-cryptsetup-generator` parses `/proc/cmdline` and `/etc/crypttab`
    to `systemd-cryptsetup@<dev>.service`
  - `systemd-cryptsetup@<dev>.service` invokes `cryptsetup` to set up the
    block devices
- arch initramfs
  - make sure systemd, keyboard and sd-encrypt hooks are enabled
    - lvm2 as well if lvm-over-luks
  - make sure cmdline has `rd.luks.name=device-UUID=root root=/dev/mapper/root`
