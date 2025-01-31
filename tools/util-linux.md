util-linux
==========

## Overview

- <https://github.com/util-linux/util-linux>
  - `agetty`, `login`
  - `dmesg`
  - `fallocate`
  - `findmnt`
  - `flock`
  - `lsblk`, `lscpu`, `lsns`
  - `more`
  - `nsenter`, `unshare`
  - `partx`
  - `su`
  - `taskset`
  - etc.

## agetty

- agetty is started by logind
- agetty initializes tty and prompts for user name
  - runs as root
  - `open_tty`
    - opens `/dev/ttyN`
    - calls `tcgetsid` and `TIOCSCTTY` to make sure `/dev/ttyN` is the
      controlling terminal
    - closes `STDIN_FILENO` and reopens `/dev/ttyN` to make sure it is stdin
    - calls `tcsetpgrp` to make itself foreground
    - dups `STDIN_FILENO` twice, for stdout and stderr
    - if vt, sets `F_VCONSOLE` flag
    - sets `TERM` to
      - explicitly specified val, or
      - if `F_VCONSOLE`, `linux`, or
      - otherwise, `vt102`
  - `termio_init`
    - if `F_VCONSOLE`,
      - calls `setlocale` to set `LC_CTYPE` to `POSIX`
      - clears the terminal
    - else,
      - `cfsetispeed`
      - `cfsetospeed`
      - `TIOCSWINSZ` to set the window size
        - default to 80x24
  - `get_logname`
    - `do_prompt` prompts `<hostname> login: `
    - reads login name
  - `execv`s `login -- <username>`

## blkid

- blkid is not user-facing
  - use lsblk instead
- blkid prints metadata of block devices
  - it loads `/run/blkid/blkid.tab` for cached metadata
  - if root, it probes as well
  - `LIBBLKID_DEBUG=all` to see the log
- internally,
  - `--probe` or `--info` forces probing and ignores the cahce
    - `libblkid` can probe
      - `BLKID_CHAIN_SUBLKS`, fs superblock
      - `BLKID_CHAIN_TOPLGY`, dev topology (e.g., sector size)
      - `BLKID_CHAIN_PARTS`, partition table
  - `--label` or `--uuid` uses the "evaluate" api
    - `blkid_evaluate_tag` opens `/dev/disk/by-label/<label>` or
      `/dev/disk/by-uuid/<uuid>`, and prints the canonical path
  - if no device given, `blkid_probe_all` scans
    - `/proc/lvm/VGs`
    - `/dev/*ubi*`
    - `/sys/block`

## lsblk

- lsblk prints metadata of block devices
  - it scans `/sys/block` for block devices
  - it finds their dependencies (partitions, dm, etc.)
- `--list-columns` lists all available columns
  - by default, it prints
    - `NAME`, device name
    - `MAJ:MIN`, major:minor device number
    - `RM`, removable device
    - `SIZE`, size of the device
    - `RO`, read-only device
    - `TYPE`, device type
    - `MOUNTPOINT`, where the device is mounted
  - `--fs` prints fs-related columns
  - `--topology` prints topology-related columns
  - `--output-all` prints all columns

## findmnt

- findmnt pretty-prints one of the mount tab files
  - `--kernel` uses `/proc/self/mountinfo` and is the default
  - `--mtab` uses `/etc/mtab`, which is a symlink to `/proc/self/mounts`
    - this is the older format
  - `--fstab` uses `/etc/fstab`
  - `--task <pid>` uses `/proc/<pid>/mountinfo` for mount namespace
- `--list-columns` lists all available columns
  - by default, it prints
    - `TARGET`, mountpoint
    - `SOURCE`, source device
    - `FSTYPE`, filesystem type
    - `OPTIONS`, all mount options
  - `--output-all` prints all columns
- filtering
  - `--pseudo` or `--real` prints only pseudo or real filesystems
  - `--shadow` pritns only over-mounted filesystems
  - `--types` prints only specified fliesystem types

## lscpu

- `lscpu_context_init_paths`
  - `cxt->rootfs` is `/`
  - `cxt->syscpu` is `/sys/devices/system/cpu`
  - `cxt->procfs` is `/proc`
- `lscpu_read_cpulists` reads which cpus are available, online, etc.
- `lscpu_read_cpuinfo` parses `/proc/cpuinfo`
- `lscpu_read_architecture` queries arch from `uname`
- `lscpu_read_archext`
- `lscpu_read_vulnerabilities` parses
  `/sys/devices/system/cpu/vulnerabilities`
- `lscpu_decode_arm` parses arm ids
  - `arm_ids_decode` parses midr implementer and partnum
