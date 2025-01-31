procps
======

## Overview

- <https://gitlab.com/procps-ng/procps>
  - `ps`, `pidof`, `pkill`, `pgrep`
  - `uptime`, `free`
  - `kill`
  - `top`
  - `w`
  - `watch`
  - etc.
- <https://gitlab.com/psmisc/psmisc>
  - `killall`, `pstree`, `fuser`
- <https://github.com/lsof-org/lsof>
  - `lsof`

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
