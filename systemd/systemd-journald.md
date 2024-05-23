systemd-journald
================

## Overview

- journald collects logs from various sources
  - `/proc/kmsg`, this reads kernel `printk` logbuf
  - `syslog`, this writes to `/dev/log` socket which is owned by journald
  - `sd_journal_print`, this writes to `/run/systemd/journal/socket`
  - stdout/stderr of service units, which point to sockets created by journald
  - kernel audit, this reads from `AF_NETLINK`/`NETLINK_AUDIT`
  - it also collects metadata about those logs, to create structured journals
- it stores journals to `/var/log/journal/<machine-id>` (persistent) or
  `/run/log/journal/<machine-id>` (volatie)
  - when journald is running, it writes journals to `system.journal` (and
    `user-<uid>.journal`)
  - when journald decides to rotate/archive the journals, it renames the
    current journal file to `system@<seq-id>-<head-seq>-<head-time>.journal`
    - `journalctl --header` to see the mysteric numbers
  - if journald detects file corruption, it appends `~` to the filename
- when `ForwardToSyslog=yes`, journald also forwards journals to
  `/run/systemd/journal/socket/syslog` socket
  - rsyslog uses the socket rather than `/dev/log` when it exists

## `journalctl`

- by default it prints all system journals
- `-b` limits to the current boot
- `-k` limits to the kernel dmesg of the current boot
- `-u` limits to a unit
- `-t` limits to a syslog ident
- `-p` limits to a priority
  - 0 or `emerg`
  - 1 or `alert`
  - 2 or `crit`
  - 3 or `err`
  - 4 or `warning`
  - 5 or `notice`
  - 6 or `info`
  - 7 or `debug`
- `-g` greps
