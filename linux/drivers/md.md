Kernel and MD
=============

## MD

- MD stands for multiple devices
  - combine multiple physical devices into a single logical device
- `CONFIG_BLK_DEV_MD` enables RAID
  - `CONFIG_MD_RAID0` for raid 0
  - `CONFIG_MD_RAID1` for raid 1
  - etc.
- `CONFIG_BCACHE` enables bcache
  - use a (fast) block device as a cache for another (slow) block device
- `CONFIG_BLK_DEV_DM` enables DM
  - device mapper is a low-level volume manager
  - high-level volume managers such as LVM2 depend on DM
  - there are also `dm-crypt`, `dm-verity`, etc. for device-level encryption /
    verification

## DM

- DM stands for Device Mapper
- `/dev/mapper/control` is used to configure dm
- `dmsetup targets` lists supported targets
  - `linear` maps a linear range from one block device to another
  - `striped` stripes the data across physical devices
  - `error` simulates io errors (for testing)
  - `crypt` provides data encryption
    - uses kernel crypto api to encrypt and descrypt data
- `dmsetup ls --tree` lists devices and their dependencies as a tree

## LUKS

- LUKS depends on `CONFIG_DM_CRYPT` and is otherwise purely in userspace
  - using plain `dm-crypt` is not recommended
  - LUKS defines a standardized header, key-slot area, and a bulk data area
    that is at the start of the block device
- wipe partition
  - `cryptsetup open --type plain -d /dev/urandom <part> to_be_wiped`
  - `dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress`
  - `cryptsetup close to_be_wiped`
- format the partition to LUKS
  - `cryptsetup luksFormat <part>`
  - the default values are
    - `--type luks2` uses luks2
    - `--cipher aes-xts-plain64`
      - use aes for block encryption
      - use xts mode to encrypt multiple blocks
      - use plain64 for IV (initial value, a salt)
    - `--hash sha256` uses sha256 for pbkdf2 and af (anti-forensic) splitter
    - `--iter-time 2000` requires 2000ms for pbkdf2
      - this is to slow down brute-force attack
    - `--key-size 256` uses AES-256
      - note that xts doubles the key size
    - `--pbkdf argon2id` uses argon2id for pbkdf
    - `--use-urandom` uses `/dev/urandom` to generate the volume key
      - volume key, aka master key, is used to encrypt the data
      - it is not stored in plain text, but is stored encrypted in key slots
- open/close the partition
  - `cryptsetup open <part> cryptroot`
  - `mkfs.btrfs /dev/mapper/cryptroot`
  - `cryptsetup close cryptroot`

## LVM

- LVM depends on `CONFIG_BLK_DEV_DM` and is otherwise purely in userspace
  - using plain `dm-linear` is not recommended
  - LVM stores metadata on the physical device for more features
- `lvm pvscan` lists physical volumes
- `lvm vgscan` lists volume groups
- `lvm lvscan` lists logical volumes
- LVM on LUKS
  - `pvcreate /dev/mapper/cryptroot`
  - `vgcreate foo /dev/mapper/cryptroot`
  - `lvcreate -L 100%ORIGIN foo -n bar`
  - `mkfs.btrfs /dev/foo/bar`

## dm-verity

- it requires at least 10 args to create
  - `version` should be 1
  - `dev` is the path or `major:minor` of the device to be verified
  - `hash_dev` is the path or `major:minor` of the device containing hashes
  - `data_block_size` is the size of a data block
  - `hash_block_size` is the size of a hash block
  - `num_data_blocks` is the number of data blocks in `dev`
    - every data block is hashed to get a digest
    - then, every `hash_block_size / digest_size` digests are hashed to get
      another layer of digests
      - this happens recursively to form a hash tree
  - `hash_start_block` is the offset of the root hash block in `hash_dev`
    - this is useful when `dev` and `hash_dev` are the same
  - `algorithm` is the algorithm
  - `digest` is the digest of the root hash block
  - `salt` is the salt
  - optional args

## dm-init

- normally, initramfs creates DM devices for mounting
- `CONFIG_DM_INIT` parses `dm-mod.create=...` cmdline to create DM devices
- the cmdline format is `dm-mod.create=<dev1>;<dev2>;...`
  - `<dev>` is `<name>,<uuid>,<minor>,<flags>,<table1>,<table2>...`
    - `<name>` is the name of the device, such as `vroot`
    - `<uuid>` is the uuid of the device, which can be empty
    - `<minor>` is the minor number of the device
    - `<flags>` is `ro` or `rw`
  - `<table>` is `<start_sector> <num_sectors> <target_type> <target_args>`
    - `<start_sector>` is the start sector of the device
    - `<num_sectors>` is the number of sectors
    - `<target_type>` can be `linear`, `crypt`, `verity`, etc.
    - `<target_args>` is target-specific
- cros contributed `dm-init` and `dm-verity`, which also supported an older
  format
  - `dm=[<num>] <dev1> <dev2>...`
    - `<num>` is number of devices, or 1 if missing
  - `<dev>` is `<name> <uuid> <mode> [<num>] , <target1> , <target2>...`
    - `<name>` is the name of the device
    - `<uuid>` is the uuid of the device, or `none`
    - `<mode>` is `ro` or `rw`
    - `<num>` is number of targets, or 1 if missing
  - `<target>` is `<start> <length> <type> <options>`
    - `<start>` is the start sector
    - `<length>` is the sector count
    - `<type>` is `verity`, etc.
    - `<options>` is target-specific
  - `dm-verity` options
    - `version` is always 0
    - `dev` is set via `payload=`
    - `hash_dev` is set via `hashtree=`
    - `data_block_size` is always 4096
    - `hash_block_size` is always 4096
    - `num_data_blocks` is always 4096
    - `num_data_blocks` and `hash_start_block` are set via `hashstart=`
    - `algorithm` is set via `alg=`
    - `digest` is set via `root_hexdigest=`
    - `salt` is set via `salt=`
