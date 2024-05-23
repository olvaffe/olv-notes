Platform2 cryptohome
====================

## Before User Login

- a cros disk has ~12 partitions
  - <https://chromium.googlesource.com/chromiumos/docs/+/HEAD/disk_format.md>
  - <https://chromium.googlesource.com/chromiumos/platform/crosutils/+/HEAD/build_library/disk_layout_v3.json>
  - 1: STATE
    - LVM
  - 2: KERN-A, 32M
  - 3: ROOT-A, 4G
    - in the kernel cmdline
      - `dm=` sets up a dm device with dm-verity
      - `root=` points to the dm device
    - the ext2 feature flags have an unknown bit set, forcing the kernel to
      mount it ro
    - `make_dev_ssd.sh -r` changes both
  - 4: KERN-B, 32M
  - 5: ROOT-B, 4G
  - 6: KERN-C, reserved
  - 7: ROOT-C, reserved
  - 8: OEM, 4MB
    - OEM web pages, links, themes, etc.
  - 9: MINIOS-A, 128MB
    - <https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/minios/README.md>
  - 10: MINIOS-B, 128MB
  - 11: POWERWASH-DATA, 4MB
    - enterprise rollback data, etc.
  - 12: EFI-SYSTEM, 64MB
    - grub2 and syslinux
- LVM
  - `lvm pvdisplay` shows one PV for partition 1
  - `lvm vgdisplay` shows one VG for the PV
  - `lvm lvdisplay` shows 19 LVs
    - `thinpool` allows overcommitting, where combined size of LVs is larger
      than the size of PV
    - `unencrypted` is mounted to `/mnt/stateful_partition`
    - `encstateful` is mounted to `/mnt/stateful_partition/encrypted`
      indirectly via `dm-crypt`
    - `dlc_*` are mounted to `/run/imageloader/*-dlc/package` indirectly via
      `dm-verity`
    - `cryptohome-<hash>-*`
- `dmsetup info` and `dmsetup status`
  - most of them are created by LVM
  - some map to dlc LVs using `dm-verity`
  - `encstateful` maps to `encstateful` LV using `dm-crypt`
- `losetup`
  - there are loop devices for squashfs images under `/usr/share`
  - there are also loop devices for some dlc LVs, to make them ro
- `findmnt`
  - `unencrypted` LV is mounted to `/mnt/stateful_partition`
    - its `/home` is bind-mounted to `/home`
    - its `/dev_image` is bind-mounted to `/usr/local`
      - it can be populated by `dev_install` which installs dev packages to
        `/usr/local`
    - its `/var_overlay/db/pkg` is bind-mounted to `/var/db/pkg`
    - its `/var_overlay/lib/portage` is bind-mounted to `/var/lib/portage`
    - its `/var_overlay/cache/dlc-images` is bind-mounted to `/var/cache/dlc-images`
  - `encstateful` LV is mounted indirectly (`dm-crypt`) to
    `/mnt/stateful_partition/encrypted`
    - its `/chronos` is bind-moutned to `/home/chronos`
    - its `/var` is bind-moutned to `/var`
  - some dlc LVs are mounted indirectly (`dm-verity`) to
    `/run/imageloader/*-dlc/package`

## `cryptohomed`

- `cryptohome` mounts user directories after user login
  - `cryptohome-<hash>-data` LV is mounted indirectly (`dm-crypt`) to
    `/home/.shadow/<hash>/mount`
    - its `/user` is bind-mounted to
      - `/home/user/<hash>`
      - `/home/chronos/u-<hash>`, for backward compat
      - `/home/chronos/user`, for backward compat
    - its `/root` is bind-mounted to `/home/root/<hash>`
    - its `/root/<daemon>` is bind-mounted to
      `/run/daemon-store/<daemon>/<hash>`
  - `cryptohome-<hash>-cache` LV is mounted indirectly (`dm-crypt`) to
    `/home/.shadow/<hash>/cache`
    - its `/root/.cache` is bind-mounted to
      - `/home/.shadow/<hash>/mount/root/.cache`
      - `/home/root/<hash>/.cache`
    - its `/root/.cache/<daemon>` is bind-mounted to
      `/run/daemon-store-cache/<daemon>/<hash>`
    - more
- IOW,
  - chrome runs as user `chronos`
  - chrome stores its system config data to `/home/chronos`
    - the dir is encrypted by some system key
    - it is mounted before user login
  - chrome stores its user config data to `/home/user/<hash>`
    - the dir is encrypted by user key
    - it is mounted after user login
  - system services store their user config data to `/home/root/<hash>`
    - the dir is encrypted by user key
    - it is mounted after user login
- previously, cryptohome uses ext4+fscrypt
  - when cryptohomed is given username/password on user login,
    - it hashes the username to get an hash
    - `/home/.shadow/<salted_hash_of_username>` is where the user data are
      - `master.0` is the encrypted vault keyset (EVK) that can be descrypted
        by password to get VK
      - `mount` is an encrypted ext4 directory and can be descrypted by VK
    - it then bind mounts various user directories
      - `/home/.shadow/<hash>/mount/user` to `/home/user/<hash>`
        - and to `/home/chronos/u-<hash>` for multi-user support
        - and to `/home/chronos/user` for backward compatibility
      - `/home/.shadow/<hash>/mount/root` to `/home/root/<hash>`
    - for a new username/password, cryptohomed creates `/home/.shadow/<hash>`
      and everything underneath
  - `/home/.shadow/<hash>/mount` uses ext4 encryption
    - file names and contents are encrypted
    - `e4crypt add_key` to add the key to the user keyring
    - everything is automatically decrypted on demand
    - it used to use stacked ecryptfs
