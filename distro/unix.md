# UNIX

## History

- Research Unix, Bell Labs
  - 1971, V1
  - 1972, V2
  - 1973, V3
  - 1973, V4
    - first version known to public
    - written in C
  - 1974, V5
    - licensed to selected educational institutions
  - 1975, V6
    - widely licensed
  - 1979, V7
    - widely licensed
  - 1985, V8
    - became internal
    - based on 4.1cBSD
  - 1986, V9
    - based on 4.3BSD
  - 1989, V10
  - 1992, Plan 9
  - 1997, Inferno
- UNIX System V, AT&T
  - 1982, System III
    - based on V7
  - 1983, SVR1
    - incorporated vi and curses from 4.1BSD
  - 1984, SVR2
  - 1987, SVR3
  - 1988, SVR4
    - incorporated 4.3BSD
  - 1992, SVR4.2 / UnixWare, SCO
  - 1987, SVR5 / UnixWare 7, SCO
- BSD, UC Berkeley
  - 1978, 1BSD
    - as an add-on to V6
  - 1979, 2BSD
  - 1979, 3BSD
    - as an add-on to V7
  - 1980, 4BSD
  - 1981, 4.1BSD
  - 1983, 4.2BSD
  - 1986, 4.3BSD
  - 1989, Net/1
    - replaced some proprietary code
  - 1991, Net/2
    - replaced more proprietary code
  - 1994, 4.4BSD-Lite
- GNU
  - 1984, GNU's Not Unix!
    - reimplement unix in copyleft license
  - <https://en.wikipedia.org/wiki/List_of_GNU_packages>

## Standardization

- 3 groups competed to be the standard
  - 1984, X/Open
  - 1988, OSF
  - 1988, UI
- 1993, COSE
  - merge of OSF and UI
  - created SUS (Single UNIX Specification)
- 1996, Open Group
  - merge of X/Open and COSE
  - controls POSIX and SUS

## Power Off

- power off process in early days
  - `wall` sends messages to ask users to logout
  - `touch /etc/nologin` blocks new logins
  - `kill -TERM` followed by `kill -KILL` kills processes
  - `sync; sync; sync` flushes buffers
  - physically power off the device
- 1975, Unix V6, `reboot` command
  - instead of physically power off, `reboot` can be the last step to trap
    back to the bootloader
- 1980, 4BSD, `halt` command and `shutdown` script
  - `halt` command flushes buffers and traps cpu to halt
  - `shutdown` sends messages, touches nologin, kills processes, and `halt`
  - they are both followed by physical poweroff
    - `halt` is for immediate poweroff
    - `shutdown` is for graceful poweroff
- 1983, SVR1, `shutdown` command
  - `shutdown` tells `init` to enter runlevel 1
    - this stops all processes orderly and `sh`
  - `shutdown -h ` tell `init` to enter runlevel 0
    - this stops all processes orderly and `halt`
  - `shutdown -r ` tell `init` to enter runlevel 6
    - this stops all processes orderly and `reboot`
- 1993, sysvinit 2.5, `poweroff` command
  - `poweroff` command flushes buffers and asks APM to power off
    - it was introduced much later after there was finally APM
  - `shutdown -P` tell `init` to enter runlevel 0, but to `poweroff` instead
    of `halt`
- IOW, `shutdown` covers the whole process
  - `halt` is the last step to halt the cpu
  - `reboot` is the last step to reboot the machine
  - `poweroff` is the last step to poweroff the machine
- systemd
  - `systemctl halt` enters `halt.target` after reaching `final.target`
    - `systemd-shutdown` calls `reboot(LINUX_REBOOT_CMD_HALT)` to halt
    - this halts the cpu and does not talk to acpi to reboot/poweroff
  - `systemctl reboot` enters `reboot.target` after reaching `final.target`
    - `systemd-shutdown` calls `reboot(LINUX_REBOOT_CMD_RESTART)` to reboot
  - `systemctl poweroff` enters `poweroff.target` after reaching `final.target`
    - `systemd-shutdown` calls `reboot(LINUX_REBOOT_CMD_POWER_OFF)` to poweroff
  - `shutdown` defaults to `systemctl poweroff`
