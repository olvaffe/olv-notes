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
  `/run/systemd/journal/syslog` socket
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

## Traditional Logging

- log producers
  - some use syslog and write to `/dev/log`
  - some write to `/var/log/*` directly
  - some writes to stdio
    - one can use `logger` to pipe the logs to syslog
- systemd-journald
  - it collects logs from various sources and saves them to `/var/log/journal`
    - including creating `/dev/log` socket to receive syslog
- rsyslog
  - it creates `/dev/log` socket to receive syslog
    - when coexisting with systemd-journald, it can be a downstream
      - systemd-journald creates `/dev/log`, and forwards logs to
        `/run/systemd/journal/syslog`
      - rsyslog recives syslog from `/run/systemd/journal/syslog`
  - `/etc/rsyslog.conf` is the config file
    - `kern.* /var/log/messages` logs kernel messages to `/var/log/messages`
    - `*.info /var/log/messages` logs INFO or higher to `/var/log/messages`
    - `authpriv.* /var/log/secure` logs auth messages to `/var/log/secure`
    - `cron.* /var/log/cron` logs cron messages to `/var/log/cron`
- logrotate
  - it is typically invoked by cron daily to rotate log files
  - `/etc/logrotate.conf` is the config file
    - e.g., rotate `/var/log/messages`
      - `weekly`
      - `rotate 5` keeps 5 rotations
      - `postrotate /usr/bin/killall -HUP syslogd endscript`
        - after rotation, sends `HUP` to syslogd so that log files are
          reopened
        - that is, rsyslog continues to write to `/var/log/messages.1` after
          rotation and the signal tells rsyslog to close and open
          `/var/log/message`
