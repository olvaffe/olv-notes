Base Utils
==========

## Base Utils

- this covers a subset of arch linux `base` dependencies
  - `coreutils`, <https://git.savannah.gnu.org/cgit/coreutils.git>
  - `file`, <https://github.com/file/file>
  - `findutils`, <https://git.savannah.gnu.org/cgit/findutils.git>
  - `gawk`, <https://git.savannah.gnu.org/cgit/gawk.git>
    - `awk`
  - `iproute2`, <https://git.kernel.org/pub/scm/network/iproute2/iproute2.git>
  - `iputils`, <https://github.com/iputils/iputils>
  - `procps-ng`, <https://gitlab.com/procps-ng/procps>
    - `ps`, `pidof`
    - `uptime`, `free`
  - `psmisc`, <https://gitlab.com/psmisc/psmisc>
  - `shadow`, <https://github.com/shadow-maint/shadow>
    - `useradd`
  - `util-linux`, <https://github.com/util-linux/util-linux>
    - `agetty`, `login`
- misc other utils
  - `lsof`, <https://github.com/lsof-org/lsof>

## uptime

- uptime returns 4 different info
  - current time
  - uptime
  - user count
  - load average
- current time
  - `time` returns `time_t` which is seconds since epoch
  - `localtime_r` converts `time_t` to `struct tm`, which is a user-friendly
    format of the specified time
- uptime
  - `procps_uptime` reads `/proc/uptime` which prints 2 numbers
    - the first number is how long in seconds the system has been up
    - thie second number is how long each core has been idle
      - on smp, it can be larger than uptime
- user count
  - if systemd, `sd_booted` returns true and `sd_get_sessions` returns the
    count of current login sessions
  - otherwise, it uses `setutent`, `getutent`, and `endutent`
    - they read `/var/run/utmp` which is maintained by `init`, `agetty`,
      `login`, etc.
    - `man 5 utmp`
- load average
  - `procps_loadavg` reads `/proc/loadavg` which returns
    - the load average of past 1, 5, and 15 minutes
    - running/total processes
    - last created pid

## free

- `procps_meminfo_new` reads `/proc/meminfo`

## pidof

- it calls `procps_pids_new` with 3 items
  - `PIDS_ID_PID`
  - `PIDS_CMD`
  - `PIDS_CMDLINE_V`
- it calls `procps_pids_get` to read `/proc/*`
  - `PIDS_ID_PID` maps to `tid` which is from `strtoul(ent->d_name, NULL, 10)`
  - `PIDS_CMD` maps to `cmd` which is from `stat`
  - `PIDS_CMDLINE_V` maps to `cmdline_v` which is from `cmdline`
- it matches against `/proc/*/cmdline` mostly
  - with `-w`, it matches against `/proc/*/stat` as well which is for kernel
    workers

## ps

- `ps -f --ppid 2 --deselect`

## useradd

- `/etc/passwd` has seven fields
  - login name
  - optional encrypted password
    - if empty, no password is required
    - if `x`, the password is stored in `/etc/shadow` instead
    - otherwise, the password is encrypted with `crypt`
      - if it is not a valid output of `crypt`, such as `*` or `!`, it is
        impossible to login with a password
  - numerical user id
  - numerical group id
  - description (e.g., real name or comment)
  - user home directory
    - `login` sets `$HOME` to the value and `chdir`s to it
  - optional user command interpreter
    - `login` assumes `/bin/sh` if empty
    - `login` sets `$SHELL` to the value and `exec`s it
- `/etc/shadow` has nine fields
  - login name
  - encrypted password
  - date of last password change
    - `0` means the password should be changed on next login
  - minimum password age
  - maximum password age
  - password warning period
  - password inactivity period
  - account expiration date
  - reserved field
- files/directories potentially modified/created by `useradd`
  - `/home/$USER`
  - `/var/mail/$USER`
  - `/etc/passwd` and `/etc/shadow`
  - `/etc/group` and `/etc/gshadow`
  - `/etc/subuid` and `/etc/subgid`
- commonly used options
  - `-b`/`-d` for base/home directory
  - `-F` for subuid/subgid for system account
  - `-G` for supplementary groups
  - `-m` for home creation
  - `-r` for system account
    - the uid will be between `SYS_UID_MIN` and `SYS_UID_MAX` defined in
      `/etc/login.defs`
    - no subuid/subgid unless `-F`
    - no home dir unles `-m`
  - `-s` for shell, default to `SHELL` defined in `/etc/default/useradd`
- system account
  - normal accounts and system accounts are no different to most tools
    except for those who choose to distinguish them
    - systemd uses `uid_is_system` to distinguish them
  - system accounts usually do not need a home directory and do not need a
    shell
    - `useradd -r -d /nonexistent -s /usr/sbin/nologin <name>`
    - if a home directory is desired, `/var/lib/<name>` is one option
  - rootless podman containers are a bit ambiguous
    - they should be run by system accounts
    - but they usually need home dirs and `XDG_RUNTIME_DIR`, where the latter
      is usually created by `pam_systemd` during normal login
    - `useradd -r -F -m <name>`
    - `loginctl enable-linger <name>` to start the user instance on boot

## awk

- awk scans over a text file
  - it treats each line as a record
  - it breaks each record into fields
  - awk 'END{print NR}' is the same as `wc -l`
    - `NR` stands for number of record
- `awk KEYWORD{ACTIONS}`
  - keywords
    - if `BEGIN`, `ACTIONS` are executed before the first record
    - if `END`, `ACTIONS` are executed after the last record
    - if `/regex/`, `ACTIONS` are executed after each record matched by
      `regex`
    - if omitted, `ACTIONS` are executed after each record
  - built-in variables
    - `$0` is the record
    - `$1` and so on are the fields
  - built-in functions
    - `print`
    - `printf`

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
  - it calls `getpwnam_r` to look up the user from `/etc/passwd`
- it calls `initgroups` to init the supplementary groups
- `loginpam_session`
  - `pam_setcred(pamh, PAM_ESTABLISH_CRED);`
    - this calls into modules such that modules can retrieve the credential
      from the core
  - session: `pam_open_session(pamh, 0);`
    - to denote the start of a session
    - modules can be various things (before changing uid/gid)
  - `pam_setcred(pamh, PAM_REINITIALIZE_CRED);`
- logging
  - `log_utmp` updates `/var/run/utmp` and `/var/log/wtmp`
  - `log_audit` logs to audit
  - `log_lastlog` logs to `_PATH_LASTLOG` (`/var/log/lastlog`) and prints
    `Last login: <time>`
- `chown_tty` changes ownership of `/dev/ttyN`
- it calls `setgid` to change the gid
- `init_environ` initializes env
  - `HOME`, `USER`, are `SHELL` are from passwd
  - `TERM` is from agetty and is preserved
  - `PATH` is set to `_PATH_DEFPATH` (`/usr/local/bin:/usr/bin`)
  - `MAIL` is set to `_PATH_MAILDIR/<username>` (`/var/spool/mail`)
  - `LOGNAME` is set to `USER`
    - `su` may change `USER` and `LOGNAME` is never changed
  - env from pam is retrieved from `pam_getenvlist`
    - the app and the modules may call `pam_putenv()` to add/delete/modify
      environment variables attached to pam handle
    - this function dumps the attached env variables
- `display_login_messages` shows motd and check `MAIL` for new mails if
  enabled
- `fork_session`
  - it detaches the tty with `TIOCNOTTY`
  - the parent is kept alive with uid 0 and gid of the user.  It `wait`s
    for the child and calls
    - `pam_setcred(cxt->pamh, PAM_DELETE_CRED)`
    - `pam_end(cxt->pamh, pam_close_session(cxt->pamh, 0))`
  - the child calls
    - `setsid` to start a session
    - `open_tty` to open tty and dups it to stdin/stdout/stderr
    - `TIOCSCTTY` to make the tty the the controlling terminal
- the child goes on and...
  - `setuid` to set the uid
  - `chdir` to `HOME`
  - `pam_end` to end pam
  - `execvp` to invoke the shell
  - `execle(shell)`

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
