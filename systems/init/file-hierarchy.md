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

## `/`

- these subdirectories or symbolic links are required
  - `bin`: Essential command binaries
    - commands desirable for single-user/maintenance mode
  - `boot`: Static files of the boot loader
    - everything required to boot the kernel
  - `dev`: Device files
  - `etc`: Host-specific system configuration
  - `lib`: Essential shared libraries and kernel modules
    - libraries needed by `bin` or `sbin`
    - kernel modules
  - `media`: Mount point for removable media
    - with subdirectories as mountpoints for removable media
    - accessible by all users
  - `mnt`: Mount point for mounting a filesystem temporarily
  - `opt`: Add-on application software packages
    - apps not managed by system package manager
    - each app should be in its subdirectory
  - `run`: Data relevant to running processes
    - must be empty on boot
    - system apps are recommended to use subdirectories
  - `sbin`: Essential system binaries
    - admin commands desirable for single-user/maintenance mode
  - `srv`: Data for services provided by this system
    - `www` if the system is a www server
    - `git` if the system is a git server
  - `tmp`: Temporary files
    - may or may not be persistent
    - visible by all users, unless chmod is used
  - `usr`: Secondary hierarchy
  - `var`: Variable data
- these subdirectories or symbolic links are optional
  - `home`: User home directories (optional)
  - `lib<qual>`: Alternate format essential shared libraries (optional)
    - e.g., `lib64` and `lib32`, with `lib` being a symbolc link to one of
      them
  - `root`: Home directory for the root user (optional)

## `/usr`

- shareable: multiple systems can share the same `/usr`
- read-only: only static data/binaries
  - can be mounted read-only other than when performing system update
- these subdirectories or symbolic links are required
  - `bin`: Most user commands
  - `lib`: Libraries
  - `local`: Local hierarchy (empty after main installation)
    - manually installed packages
    - separated out from `/usr` to avoid overwritten by system update
  - `sbin`: Non-vital system binaries
  - `share`: Architecture-independent data
- these subdirectories or symbolic links are optional
  - `games`: Games and educational binaries (optional)
  - `include`: Header files included by C programs
  - `libexec`: Binaries run by other programs (optional)
    - internal commands of other commands/libraries
    - not intended for direct invocation
  - `lib<qual>`: Alternate Format Libraries (optional)
  - `src`: Source code (optional)
    - for reference only, not for building

## `/var`

- variable data files
  - packages are installed to /usr and are static
  - persistent data created by packages goes to /var
    - e.g., logs, spools, package manager database, databases
- these subdirectories or symbolic links are required
  - `cache`: Application cache data
    - persistent, but apps must be able to regenerate the data in this
      directory when it is cleared
  - `lib`: Variable state information
    - apps must use `/var/lib/<name>` subdirectory or `/var/lib/misc`
  - `local`: Variable data for /usr/local
  - `lock`: Lock files
  - `log`: Log files and directories
  - `opt`: Variable data for /opt
    - opt apps must use `/var/opt/<name>` for their variable data
  - `run`: Data relevant to running processes
    - use `/run` instead
  - `spool`: Application spool data
    - data awaiting processing
  - `tmp`: Temporary files preserved between system reboots
- these subdirectories or symbolic links are optional
  - `account`: Process accounting logs (optional)
  - `crash`: System crash dumps (optional)
  - `games`: Variable game data (optional)
  - `mail`: User mailbox files (optional)
  - `yp`: Network Information Service (NIS) database files (optional)

## Linux-specific

- `/proc`
- `/sys`

## XDG Base Directory Specification

- <https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html>
  - freedesktop.org was formerly known as X Desktop Group, XDG
- `XDG_DATA_HOME`
  - defaults to `$HOME/.local/share`
- `XDG_CONFIG_HOME`
  - defaults to `$HOME/.config`
- `XDG_STATE_HOME`
  - defaults to `$HOME/.local/state`
- `XDG_CACHE_HOME`
  - defaults to `$HOME/.cache`
- `XDG_DATA_DIRS`
  - defaults to `/usr/local/share/:/usr/share/`
  - additional data search paths
- `XDG_CONFIG_DIRS`
  - defaults to `/etc/xdg`
  - additional config search paths
- `XDG_RUNTIME_DIR`
  - no default
  - if exists, the directory must be owned by user and has access mode 0700

## `pam_systemd`

- `man pam_systemd`
  - `XDG_SESSION_ID`
  - `XDG_RUNTIME_DIR`
  - `XDG_SESSION_TYPE`
  - `XDG_SESSION_CLASS`
  - `XDG_SESSION_DESKTOP`
  - `XDG_SEAT`
  - `XDG_VTNR`
