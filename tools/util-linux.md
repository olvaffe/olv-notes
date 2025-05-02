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

## login

- `/etc/login.defs` is the config file
  - it is shared by multiple tools and only a subset of configs are applicable
    to login
- `login` is started by agetty and prompts for password
- `init_tty`
  - it closes the current stdin/stdout/stderr and calls `vhangup`
  - it gets the terminal path, `/dev/ttyN`, via `ttyname(STDIN_FILENO)`
  - it opens the terminal and dups the fd to stdin/stdout/stderr
- `init_loginpam`
  - `pam_start("login", username, &conv, &pamh);`
    - `username` is specified by `agetty`
    - `conv` is `misc_conv` from `libpam_misc`
    - this function allocates a `pam_handle_t` and reads both
      - `/etc/pam.d/<service>`
        - `<service>` is `login` in this case
        - these rules are added to `pamh->handlers.conf`
      - `/etc/pam.d/other`
        - these rules are added to `pamh->handlers.other`, which is used only
          when `pamh->handlers.conf` is empty
    - it `dlopen()` the necessary modules
  - it passes information for PAM modules
    - `pam_set_item(pamh, PAM_RHOST, NULL);`
    - `pam_set_item(pamh, PAM_TTY, tty);`
      - tty is from `ttyname(0)` so is normally `/dev/ttyN`
    - `pam_set_item(pamh, PAM_USER_PROMPT, "<hostname> login:");`
- `loginpam_auth`
  - auth: `pam_authenticate (pamh, 0);`
    - it is called up to `LOGIN_MAX_TRIES` (3) times
    - if it fails for the first time, the user name is cleared with
      `pam_set_item(pamh, PAM_USER, NULL)` such that pam prompts for a new
      user name
    - when modules call `pam_get_user()`, and there is no `PAM_USER`, it
      starts a `PAM_PROMPT_ECHO_OFF` conversation with the app to get the user
      name using `PAM_USER_PROMPT` as the prompt.  The result is set to
      `PAM_USER` for later use.
    - modules start a `PAM_PROMPT_ECHO_OFF` conversation with the app to get
      the password as well.  For `pam_unix`, the prompt is localized
      `Password:`.  The password is set to `PAM_AUTHTOK` for later use.
    - the auth token for the user is then verified
- `loginpam_acct`
  - account: `pam_acct_mgmt (pamh, 0);`
    - it returns `PAM_SUCCESS` if the account is valid and active
    - if the account requires to update its password, `PAM_NEW_AUTHTOK_REQD` is
      returned
      - the app should call `pam_chauthtok(pamh, PAM_CHANGE_EXPIRED_AUTHTOK);`
        to update the password
  - the app retrieves the user name back from `PAM_USER`
- `xgetpwnam`

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
