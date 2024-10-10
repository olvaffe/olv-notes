Debian
======

## stable releases

- schedule
  - debian has a stable release roughly every 2 years
  - a stable release has a point release roughly every 2 months
  - after a new stable release, the old stable release has security support
    for 1 year
- suites/repos
  - a release uses `stable` repo
    - use codenames such as `bookworm` instead to avoid unexpected upgrade to
      a new release
  - updates are staged and tested in `stable-proposed-updates` until a point
    release is made
    - at which point, the updates are reflected in `stable`
  - updates that are security-releated are also added to `stable-security`
  - updates that are time-sensitive (e.g., timezone data) are also added to
    `stable-updates`
  - backports from `testing` are in `stable-backports`
    - it consists of selected packages from `testing` which are re-compiled
      for `stable`
- components
  - `main` consists of DFSG-compliant packages
  - `contrib` consists of DFSG-compliant packages whose dependencies are not
    in `main`
  - `non-free` consists of non-DFSG-compliant packages
  - `non-free-firmware` consists of non-DFSG-compliant firmware packages
- kernel
  - stable releases use lts kernels
  - 10, buster, 4.19
  - 11, bullseye, 5.10
  - 12, bookworm, 6.1
  - 13, trixie, 6.??
- bookworm
  - `deb http://deb.debian.org/debian bookworm main non-free-firmware`
  - `deb http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware`
  - `deb http://deb.debian.org/debian bookworm-updates main non-free-firmware`

## Initial Setup

- packages
  - `apt-mark` packages appropriately
  - boot: `init linux-image-arm64`
  - admin
    - `locales man-db sudo systemd-resolved systemd-zram-generator vim whiptail`
    - `dosfstools fdisk`
    - `auditd logrotate unattended-upgrades`
  - network
    - `iproute2 iputils-ping nftables wireguard`
    - `wireless-regdb wpasupplicant` (or `iwd`)
    - `ssh msmtp`
  - container
    - `containers-storage podman`
  - rpi
    - `bluez-firmware firmware-brcm80211 raspi-firmware`
    - remove locally-installed `*rpi*` and `*raspi*` files under `/etc`,
      `/usr/local`, and `/boot/firmware`
  - gce
    - `grub-cloud-amd64 linux-image-cloud-amd64`
    - `google-compute-engine-oslogin google-guest-agent`
    - `google-cloud-sdk google-compute-engine google-osconfig-agent`
  - extra
    - `file git lsof strace`
    - `pciutils usbutils`
    - `curl`
- base setup
  - `dpkg-reconfigure locales tzdata`
  - `systemctl enable --now systemd-networkd systemd-resolved`
  - `visudo`
  - `useradd -m olv`

## APT

- `apt-get`
  - `update` updates package index
  - `upgrade` updates all packages that have newer versions
  - `dist-upgrade` is similar to `upgrade` but may install/remove new packages
  - `install` installs listed packages
  - `reinstall` is the same as `install --reinstall`
  - `remove` removes listed packages
  - `pruge` is similar to `remove` and pruges config files
  - `source` downloads the sources of listed packages
  - `build-dep` installs build dependencies of listed packages
  - `downloads` downalods the listed packages
  - `clean` removes all cached packages
  - `autoclean` removes cached packages that are outdated
  - `autoremove` removes packages that were installed as dependencies but are
    not dependencies anymore
- `apt-cache`
  - `show` shows info of listed packages
  - `search` searches matching packages
  - `depends` lists dependeices of listed packages
  - `rdepends` lists reverse-dependeices of listed packages
- `apt-mark`
  - `auto` and `manual` mark the listed packages as automatically/manually
    installed respectively
  - `showauto` and `showmanual` list automatically/manually installed packages
    respectively
  - `minimize-manual` marks dependencies of meta packages as automatically
    installed
- to clean up packages,
  - `apt-mark showmanual | xargs sudo apt-mark auto` to mark everything auto
  - `apt autoremove --dry-run` to selectively mark packages manual
  - `apt autoremove` to remove unneeded ones
  - this ignores packages that are on `Recommends` or `Suggests`
    - `-o APT::Autoremove::RecommendsImportant=0` to remove recommends
    - `-o APT::Autoremove::SuggestsImportant=0` to remove suggests
    - A recommends B when A is pointless without B for most users
    - A suggests B when A is enhanced by B
- to find orphaned files,
  - `cat /var/lib/dpkg/info/*.list | sed -e 's,^/\(bin\|lib\|sbin\),/usr/\1,' | sort | uniq > dpkg.list`
  - `find / -xdev -path "/home/*" -prune -o -print | sort > find.list`

## dpkg

- `dpkg`
  - `-i <pkg>.deb` installs `<pkg>.deb`
  - `-r <pkg>` removes an installed package
  - `-P <pkg>` purges an installed package
  - `-L <pkg>` lists the contents an installed package
  - `-l` lists installed packages
  - `-S <path>` lists packages who own `<path>`
- `dpkg-deb`
  - `-c <pkg>.deb` shows the contents of `<pkg>.deb`
  - `-x <pkg>.deb <dir>` extracts `<pkg>.deb` to `<dir>`

## deb

- a .deb file is an ar-archive
- `ar t <pkg>.deb` shows 3 files
  - `debian-binary` specifies the package format version (2.0)
  - `control.tar.xz` contains package metadata and install scripts
  - `data.tar.xz` contains files
- `ar x <pkg>.deb` to extracts them

## `debootstrap`

- `debootstrap` does 4 things
  - download deb packages
  - unpack them into the target directory
  - chroot into the target directory
    - this requires root
  - run the installation and configuration scripts from the packages
    - this forbids bootstrapping another architecture
- `sudo debootstrap stable stable-chroot`
  - or replace `sudo` by `fakechroot fakeroot`

## cross-`debootstrap`

- `apt install qemu-user-static`
  - if you mess up somewhat, re-enable binfmts
    - `sudo update-binfmts --disable`
    - `sudo update-binfmts --enable`
- `debootstrap --arch arm64 stable stable-arm64-chroot`
- `chroot stable-arm64-chroot /bin/bash -i`
  - or, to run it as a container,
    - `chown -R 65536.65536 stable-arm64-chroot`
    - `systemd-nspawn -UD stable-arm64-chroot`
- old way
  - `apt install qemu-user-static`
  - `debootstrap --arch arm64 --foreign stable stable-arm64-chroot`
    - `--foreign` skips the last two steps of `debootstrap`
  - `cp /usr/bin/qemu-aarch64-static stable-arm64-chroot`
  - `chroot stable-chroot /qemu-aarch64-static /bin/bash -i`
  - `/debootstrap/debootstrap --second-stage`

## cross-compile

- `apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu`
- in chroot, install whatever dependent -dev packages
- note that some -dev packages in chroot use absolute links
  - `ls -l /usr/lib/aarch64-linux-gnu | grep ' -> /lib'`
  - they need to be fixed otherwise the linker can unexpectedly fall back to
    the static libraries

## Toolchain Packages

- `binutils` and dependencies
  - `binutils` provides `/usr/bin/{ld,ld.bfd,ld.gold,nm,readelf,...}`
    symblinks
  - `binutils-x86-64-linux-gnu` provides
    `/usr/bin/x86_64-linux-gnu-{ld.bfd,ld.gold,nm,readelf,...}` binaries
- `cpp-14` and dependencies
  - `cpp-14` provides `/usr/bin/cpp-14` symlink
  - `cpp-14-x86-64-linux-gnu` provides `/usr/bin/x86_64-linux-gnu-cpp-14` binary
- `gcc-14` and dependencies
  - `gcc-14` provides `/usr/bin/gcc-14` symlink
  - `gcc-14-x86-64-linux-gnu` provides `/usr/bin/x86_64-linux-gnu-gcc-14`
    binary
    - `binutils-x86-64-linux-gnu`
    - `cpp-14-x86-64-linux-gnu`
    - `libgcc-14-dev` provides gcc runtime, `/usr/lib/gcc/x86_64-linux-gnu/14`
    - `libc6` provides c runtime,
      `/usr/lib/x86_64-linux-gnu/{ld-linux-x86-64,libc,libm,...}.so.*`
    - `libstdc++6` provides c++ runtime, `/usr/lib/x86_64-linux-gnu/libstdc++.so.6`
- `g++-14` and dependencies
  - `g++-14` provides `/usr/bin/g++-14` symlink
  - `g++-14-x86-64-linux-gnu` provides `/usr/bin/x86_64-linux-gnu-g++-14`
    binary
    - `gcc-14-x86-64-linux-gnu`
    - `libstdc++-13-dev` provides c++ runtime
- `gcc` provides `/usr/bin/gcc` symlink
- `g++` provides `/usr/bin/g++` symlink
- aarch64 cross-compiler is similar
  - `binutils-aarch64-linux-gnu`
  - `cpp-14-aarch64-linux-gnu`
  - `gcc-14-aarch64-linux-gnu`
  - `g++-14-aarch64-linux-gnu`
  - except
    - `x86_64-linux-gnu` becomes `aarch64-linux-gnu`
    - `/usr/lib/gcc` becomes `/usr/lib/gcc-cross`
    - `/usr/libexec/gcc` becomes `/usr/libexec/gcc-cross`
    - there is a mini sysroot at `/usr/aarch64-linux-gnu`
- i686 cross-compiler similar
  - `binutils-i686-linux-gnu`
  - `cpp-14-i686-linux-gnu`
  - `gcc-14-i686-linux-gnu`
  - `g++-14-i686-linux-gnu`
  - there are also multilib (cross-compile for 32-bit on 64-bit)
    - `gcc-13-multilib-i686-linux-gnu` reuses the i686 cross-compiler
    - `gcc-13-multilib` is the older way and conflicts with all other
      cross-compilers

## Minimum Package

- minimum packaging files
  - `debian/changelog`
    - `man deb-changelog`
    - `debchange --create --package <name> --version <ver>`
  - `debian/source/format`
    - `man dpkg-source`
    - `mkdir debian/source`
    - `echo '3.0 (native)' > debian/source/format`
  - `debian/control`
    - `man deb-src-control`
    - source package stanza
      - `Source: <name>`
      - `Maintainer: <author> <email>`
      - `Build-Depends: debhelper-compat (= 13)`
    - binary package stanza
      - `Package: <name>`
      - `Architecture: all`
  - `debian/rules`
    - `man deb-src-rules`
    - `vi debian/rules`
      - `#!/usr/bin/make -f`
      - `%:`
      - `	dh $@`
    - `chmod +x debian/rules`
- build source and binary packages
  - `dpkg-buildpackage --no-sign`
- fix warnings
  - `debian/control` source package stanza
    - `Standards-Version: 4.6.2`
    - `Section: metapackages`
    - `Priority: optional`
  - `debian/control` binary package stanza
    - `Description: <desc> (metapackage)`
- fix lintian
  - `debian/control` binary package stanza
    - `Description: <desc> (metapackage)\n <extended-desc>`
  - `debian/copyright`
    - <https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/>
    - header stanza
      - `Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/`
    - files stanza
      - `Files: *`
      - `Copyright: <year> <name>`
      - `License: MIT`
- build with equivs
  - `echo "Version: <ver>" >> debian/control`
  - `equivs-build debian/control`

## Logs

- `update-alternatives` logs to `/var/log/alternatives.log`
- `apt` logs to `/var/log/apt`
  - `term.log` is stdout
  - `history.log` is the package install/remove/upgrade history
- `dpkg` logs to `/var/log/dpkg.log`
  - it is low-level than apt
  - a package change done by `dpkg` will not show up in apt logs
- `unattended-upgrades` logs to `/var/log/unattended-upgrades`
  - `unattended-upgrades.log` contains the invocation log (no update, update
    package X)
  - `unattended-upgrades-dpkg.log` contains dpkg stdout