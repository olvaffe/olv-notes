Filesystem Hierarchy
====================

## systemd

- `man 7 file-hierarchy`
- `systemd-path`
- on EFI systems, the ESP partition is commonly mounted to `/boot`
- `/run/user/<uid>`, emptied when reboot or user logs out
  - `$XDG_RUNTIME_DIR` should point to this directory
- user packages
  - there was system-managed packages, locally installed packages
    (`/usr/local`), and opt packages (`/opt/<name>`)
  - user packages go to `~/.local`
    - originated from <https://www.python.org/dev/peps/pep-0370/>

## Storage Planning

- static vs variable
  - most dirs are static except for system update
  - some dirs are pseudo mountpoints
  - these dirs can be considered static
    - `/etc` if we consider config update as system update
    - `/root` if we use it only for system update
  - these dirs are variable
    - `/home` contains user home dirs
    - `/srv` contains served data
    - `/var` contains app data
- `/`
  - on workstations, 16G suffices
  - on servers, 4G suffices
  - speed matters
- `/home`
  - on workstations with real users, both capacity and speed matter
  - on servers, it can be considered static if logged in only for system update
    - and if rootless podman uses `/var/lib/pod-foo`
- `/srv`
  - on workstations, it is unused
  - on servers, capacity matters if serving tons of data
    - data are served with minimal processing so we want io bw to exceed net
      bw as well
- `/var`
  - logs, caches, spools can grow reasonably
  - on servers, some services can have huge app data
    - databases
    - containers
    - vms

## FHS

- <https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html>
  - but we describe what modern linux distros do
- `/`
  - split `/` and `/usr` is undesirable
    - it allowed small `/` to mount `/usr`, but the role is taken by initrd
  - `/bin` is symlink to `usr/bin`
  - `/boot` is esp/xbootldr mountpoint
  - `/dev` is devtmpfs mountpoint
  - `/etc` contains configuration files
  - `/home` contains user home dirs (optional)
  - `/lib` is symlink to `usr/lib`
  - `/lib<qual>` is symlink to `usr/lib<qual>` (optional)
  - `/media` contains mountpoints for removable media
  - `/mnt` is temp mountpoint
  - `/opt` contains self-contained apps
    - `/opt/<app>` or `/opt/<vendor>/<app>`
  - `/root` is root home dir (optional)
  - `/run` is tmpfs mountpoint
  - `/sbin` is symlink to `usr/sbin`, or to `usr/bin` if merged bin/sbin
  - `/srv` contains data served by services
  - `/tmp` is tmpfs mountpoint
- `/usr`
  - `/usr/bin` contains user executables
  - `/usr/games` contains self-contained games (optional)
  - `/usr/include` contains C include files (optional)
  - `/usr/lib` contains libraries
  - `/usr/libexec` contains helper executables (optional)
    - some distros use `/usr/lib`
  - `/usr/lib<qual>` contains libraries for alternate archs (optional)
  - `/usr/local` contains locally-installed apps
    - it mirrors `/usr` hierarchy mostly, but it also has `/usr/local/etc`
  - `/usr/sbin` contains system executables, or symlink to `bin`
  - `/usr/share` contains arch-independent data
  - `/usr/src` contains C source files (optional)
- `/var`
  - by keeping variable data in `/var`, `/usr` can be read-only
  - `/var/account` is no longer used (optional)
  - `/var/cache` contains app cache
  - `/var/crash` is no longer used (optional)
  - `/var/games` contains game data (optional)
  - `/var/lib` contains app data
    - `/var/lib/<app>` or `/var/lib/misc/<app>`
  - `/var/local` contains locally-installed app data
  - `/var/lock` is symlink to `/run/lock`
  - `/var/log` contains logs
  - `/var/mail` contains user mailboxes, or symlink to `spool/mail` (optional)
  - `/var/opt` contains self-contained app data
  - `/var/run` is symlink to `/run`
  - `/var/spool` contains data to be processed
  - `/var/tmp` is temp data
    - it is cleaned occasionally between boots
  - `/var/yp` is no longer used (optional)
- linux-specific
  - `/proc` is proc
  - `/sys` is sysfs
