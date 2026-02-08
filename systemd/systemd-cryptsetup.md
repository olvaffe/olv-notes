systemd-cryptsetup
==================

## Setup

- `/etc/kernel/uki.conf`
  - `[PCRSignature:initrd]`
  - `PCRPrivateKey=/etc/systemd/tpm2-pcr-private-key.pem`
  - `PCRPublicKey=/etc/systemd/tpm2-pcr-public-key.pem`
- `ukify genkey -c /etc/kernel/uki.conf` generates the key to sign pcr
  policies
- `systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 --tpm2-public-key-pcrs=11 --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem <part>`
  adds a luks key slot protected by tpm2
  - the key slot holds tpm2-encrypted volume key for decryption
  - `cryptsetup luksDump <part>` shows
    - the key slot is no different than a password-protected key slot
    - there is a `systemd-tpm2` token associated with the key slot
  - `systemd-tpm2` token has two tpm2 policies
    - a pcr policy that requires pcr 7 to have the current value
      - pcr 7 contains the hash of secure boot state
      - tpm2 will refuse to decrypt if there is any change to secure boot
        state
    - an authorize policy that requires pcr 11 to follow any future pcr policy
      that is signed by `/etc/systemd/tpm2-pcr-private-key.pem`
      - pcr 11 contains the hash of uki image initially
      - systemd invokes `system-pcrextend` to change the value at different
        boot phases
        - e.g., it changes on `enter-initrd` and changes again on
          `leave-initrd`, to ensure a signed pcr policy only applies during
          initrd
      - uki embeds a signed pcr policy that requires pcr 11 to have a certain
        pre-calculated value
      - tpm2 will refuse to decrypt if pcr 11 does not match the
        pre-calculated value backed into uki
        - that is, while secure boot might boot a uki (as long as it is signed
          by a key in db), tpm2 decrypts the volume key only when uki embeds a
          signed pcr policy as well
- on arch, `mkinitcpio -p linux` regenerates uki
  - it invokes `ukify` to generate uki
  - `ukify` invokes `systemd-measure sign` to create a signed pcr policy and
    embeds the signed pcr policy in uki
  - the signed pcr policy requires pcr 11 to have value X, where X is
    calculated by `systemd-measure` by simulating `systemd-stub` and
    `system-pcrextend` behavior during boot
  - make sure systemd, keyboard and sd-encrypt hooks are enabled
    - lvm2 as well if lvm-over-luks
  - make sure cmdline has `rd.luks.name=device-UUID=root root=/dev/mapper/root`
- inside initrd on boot,
  - `systemd-cryptsetup-generator` parses `/proc/cmdline` and `/etc/crypttab`
    to `systemd-cryptsetup@<dev>.service`
  - `systemd-cryptsetup@<dev>.service` invokes `systemd-cryptsetup attach` to
    set up the block devices
    - `systemd-stub` has unpacked the signed pcr policy from uki to `/.extra`
    - `systemd-cryptsetup` requests tpm2 to decrypt the volume key using the
      signed pcr policy
    - thanks to boot phases, initrd is the only boot phase that the signed pcr
      policy applies (as configured in `uki.conf`)
- revisit `--tpm2-pcrs` and `--tpm2-public-key-pcrs`
  - `--tpm2-pcrs` creates a pcr policy which unseals the volume key only when
    pcrs have specific values (taken from current values)
    - the values are a part of the policy and are fixed
  - `--tpm2-public-key-pcrs` creates an authorize policy which unseals the
    volume key only when a future pcr policy allows it
    - `--tpm2-public-key` is a part of the policy and is fixed
    - the future pcr policy must have a signature verified by the public key
  - uki `.pcrsig` and `.pcrpkey` sections
    - `.pcrsig` cotains the signed pcr policy created by `systemd-measure`
    - `.pcrpkey` contains the public key to verify the signed pcr policy
    - together with `--tpm2-public-key-pcrs`, tpm2 unseals the volume key only
      when the uki is booted
- <https://0pointer.net/blog/brave-new-trusted-boot-world.html> describes the
  concepts
