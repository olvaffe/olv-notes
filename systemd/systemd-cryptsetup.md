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
- `--tpm2-pcrs` and `--tpm2-public-key-pcrs`
  - `--tpm2-pcrs` creates a pcr policy which unseals the volume key only when
    pcrs have specific values (taken from current values)
    - the values are a part of the policy and are fixed
  - `--tpm2-public-key-pcrs` creates an authorize policy which unseals the
    volume key only when a future pcr policy allows it
    - `--tpm2-public-key` is a part of the policy and is fixed
    - the future pcr policy must have a signature verified by the public key
- uki `.pcrsig` and `.pcrpkey` sections
  - `.pcrsig` cotains the expected pcr 11 value and is signed
  - `.pcrpkey` contains the public key to verify the signature
  - together, they can create a pcr policy for use with
    `--tpm2-public-key-pcrs`
- <https://0pointer.net/blog/brave-new-trusted-boot-world.html>
