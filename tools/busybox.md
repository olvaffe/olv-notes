BusyBox
=======

## busybox build system

- `scripts/gen_build_files.sh` parses the applets and generate various files
  - `//config` is combined with `Config.src` to generate `Config.in`
  - `//applet` is combined with `include/applets.src.h` to generate
    `include/applets.h`
  - `//kbuild` is combined with `Kbuild.src` to generate `Kbuild`
  - `//usage` is combined with `include/usage.src.h` to generate
    `include/usage.h`
    - there are several levels of usage docs
      - `trivial` is always included
      - `full` is included when `CONFIG_FEATURE_VERBOSE_USAGE`
      - `notes`, `example`, and `sample` are never included?

## init

- `/etc/inittab` is used by `init`
  - `::respawn:/bin/sh` spawns a shell,
    - it inherits stdin/stdout/stderr from `/init`, which is `/dev/console`
  - `ttyS0::respawn:/bin/sh` also spawns a shell
    - it opens `/dev/ttyS0` as stdin/stdout/stderr
  - `ttyS0::respawn:/sbin/getty -L 115200 ttyS0` spawns getty
- `/etc/securetty` is used by `login`
  - login refuses root unless the tty is on `/etc/securetty`
  - when `/proc` is not mounted, login cannot query the tty and uses `UNKNOWN`
    as the name
    - `echo UNKNONWN >> /etc/securetty` to work around

## toybox

- started from scratch by then busybox maintainer after licensing argument

## initramfs

- prepare cpio archive
  - `mkdir tmp && cd tmp`
  - `curl -L https://archlinux.org/packages/extra/x86_64/busybox/download | tar --zstd -x --strip 1 usr/bin/busybox`
  - `curl -L http://mirror.archlinuxarm.org/aarch64/extra/busybox-1.36.1-2-aarch64.pkg.tar.xz | tar -Jx --strip 1 usr/bin/busybox`
  - `vi init`
  - `chmod 755 init`
  - `find . | cpio -o -H newc -R root:root | zstd > ../initramfs.cpio.zst`
- a minimal busybox-based initramfs can do these in `/init` to get a shell
  - `#!/bin/busybox sh`
  - `/bin/busybox --install -s /bin`
  - `export PATH=/bin`
  - `mkdir /proc; mount -t proc none /proc`
  - `mkdir /sys; mount -t sysfs none /sys`
  - `mount -t devtmpfs none /dev`
  - `exec sh -i`
