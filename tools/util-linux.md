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
  - it runs as root
  - its goal is to init tty line, get user name interactively, and `execve` to
    `login`
- `main`
  - `update_utmp` updates utmp
    - `utmpdump /var/run/utmp` to see the entry
  - `open_tty`
    - opens `/dev/ttyN`
    - calls `isatty` to sanity check
    - calls `tcgetsid` and `TIOCSCTTY` to make sure `/dev/ttyN` is the
      controlling terminal
    - closes `STDIN_FILENO` and reopens `/dev/ttyN` to make sure it is stdin
    - calls `tcsetpgrp` to make itself foreground
    - closes `STDOUT_FILENO` and `STDERR_FILENO`, and then dups `STDIN_FILENO`
      twice for stdout and stderr
    - if vt, sets `F_VCONSOLE` flag
    - sets `TERM` to
      - explicitly specified val, or
      - if `F_VCONSOLE`, `linux`, or
      - otherwise, `vt102`
  - `termio_init`
    - if `F_VCONSOLE`,
      - calls `setlocale` to set `LC_CTYPE` to `POSIX`
      - calls `tcsetattr` to set the line
      - clears the terminal
    - else,
      - calls `cfsetispeed` and `cfsetospeed` to set the speed
      - `TIOCSWINSZ` to set the window size, defaulting to 80x24
      - calls `tcsetattr` to set the line
  - `get_logname` reads login name from stdin
    - `eval_issue_file` parses `/etc/issue`
    - `do_prompt` prompts for login name
      - `print_issue_file` prints `/etc/issue`
      - it prints `<hostname> login: `
    - reads login name
  - calls `tcsetattr` to set the line for `login`
  - `execv`s `login -- <username>`

## login

- `login` is started by agetty
  - it runs as root
  - it auths and starts user session using pam, forks a child to exec shell as
    the user, and waits for the child to exit
- `main`
  - `initialize` parses args and `/etc/login.defs`
    - `/etc/login.defs` is shared by multiple tools and only a subset of
      configs are applicable to login
  - `setpgrp` sets the process group id
  - `init_tty`
    - `get_terminal_name` queries the tty path using `ttyname(STDIN_FILENO)`
    - it closes the current stdin/stdout/stderr
    - `vhangup` hangs up the tty line
    - it reopens the tty line and dups the fd to stdin/stdout/stderr
  - `openlog` connects to syslog
  - `init_loginpam` init pam
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
      - `pam_set_item(pamh, PAM_RHOST, cxt->hostname);`
      - `pam_set_item(pamh, PAM_TTY, cxt->tty_path);`
      - `pam_set_item(pamh, PAM_USER_PROMPT, "<hostname> login:");`
        - this is for retries
  - `loginpam_auth` auths the user
    - `pam_authenticate (pamh, 0);`
      - it is called up to `LOGIN_RETRIES` times
      - if auth fails, for example due to wrong password,
        - it logs to syslog
        - `log_btmp` logs to `/var/log/btmp`
        - `log_audit` logs to audit
        - `pam_set_item(pamh, PAM_USER, NULL)` clears the user name from
          agetty, such that pam prompts for a new user name
    - pam internals
      - when modules call `pam_get_user()`, and there is no `PAM_USER`, it
        starts a `PAM_PROMPT_ECHO_OFF` conversation with the app to get the user
        name using `PAM_USER_PROMPT` as the prompt.  The result is set to
        `PAM_USER` for later use.
      - modules start a `PAM_PROMPT_ECHO_OFF` conversation with the app to get
        the password as well.  For `pam_unix`, the prompt is localized
        `Password:`.  The password is set to `PAM_AUTHTOK` for later use.
      - the auth token for the user is then verified
  - `loginpam_acct` verifies the account
    - `pam_acct_mgmt (pamh, 0);`
      - it returns `PAM_SUCCESS` if the account is valid and active
      - if the account requires its password updated, `PAM_NEW_AUTHTOK_REQD`
        is returned
        - the app should call `pam_chauthtok(pamh, PAM_CHANGE_EXPIRED_AUTHTOK);`
          to update the password
    - the app retrieves the user name back from `PAM_USER`
  - `xgetpwnam` gets the user entry from `/etc/passwd`
  - `initgroups` inits group list for the process
  - `loginpam_session` starts the user session
    - `pam_open_session` opens users session
  - `endpwent` closes `/etc/passwd`
  - `log_utmp` updates `/var/log/wtmp`
  - `log_audit` logs to audit
  - `log_lastlog` updates `/var/log//lastlog`
  - `chown_tty` changes uid/gid/mode of the tty line
  - `setgid` changes process gid
  - `init_environ` inits env
    - it sets `HOME`, `USER`, and `SHELL` from `/etc/passwd`
    - it sets `PATH` from `/etc/login.defs`
    - it sets `MAIL` to `/var/spool/mail/<user>`
    - it sets `LOGNAME` from `/etc/passwd`
    - it sets more vars from `pam_getenvlist`
  - `log_syslog` logs to syslog
  - `display_login_messages` shows motd and mail notice
  - `fork_session` forks the child process
    - `TIOCNOTTY` detaches tty
    - `closelog` closes syslog connection
    - `fork` forks the child
    - the parent `wait`s until the child exits and calls `pam_close_session`
      to close the user session
    - the child
      - `setsid` to start a new session
      - `open_tty` to reopen the tty and dups it to stdin/stdout/stderr
      - `TIOCSCTTY` to attach to tty
    - only the child returns from `fork_session`
- the child process
  - `setuid` to become the user
  - `chdir` to `HOME`
  - `pam_end` to clean up pam
  - `execvp` to `SHELL`

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
