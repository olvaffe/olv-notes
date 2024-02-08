Base Utils
==========

## Base Utils

- arch linux `base` depends on
  - `coreutils`, <https://git.savannah.gnu.org/cgit/coreutils.git>
  - `file`, <https://github.com/file/file>
  - `findutils`, <https://git.savannah.gnu.org/cgit/findutils.git>
  - `iproute2`, <https://git.kernel.org/pub/scm/network/iproute2/iproute2.git>
  - `iputils`, <https://github.com/iputils/iputils>
  - `procps-ng`, <https://gitlab.com/procps-ng/procps>
  - `shadow`, <https://github.com/shadow-maint/shadow>
  - `util-linux`, <https://github.com/util-linux/util-linux>
  - more
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
  - `useradd -r -d /nonexistent -s /usr/sbin/nologin <name>`
  - `useradd -r -b /srv -F -m <name>`
