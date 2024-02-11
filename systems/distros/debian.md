Debian
======

## stable releases

- schedule
  - debian has a stable release roughly every 2 years
  - a stable release has a point release roughly every 2 months
  - after a new release, the old release has security support for 1 year
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
- bookworm
  - `deb http://deb.debian.org/debian bookworm main non-free-firmware`
  - `deb http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware`
  - `deb http://deb.debian.org/debian bookworm-updates main non-free-firmware`

## Initial Setup

- packages
  - `apt-mark` packages appropriately
  - boot: `init linux-image-arm64`
  - admin
    - `locales man-db sudo systemd-zram-generator systemd-resolved vim whiptail`
    - `dosfstools fdisk`
  - network
    - `iproute2 iputils-ping nftables`
    - `wireless-regdb wpasupplicant` (or `iwd`)
    - `ssh`
  - container
    - `containers-storage podman`
  - rpi
    - `bluez-firmware firmware-brcm80211 raspi-firmware`
    - remove locally-installed `*rpi*` and `*raspi*` files under `/etc`,
      `/usr/local`, and `/boot/firmware`
  - extra
    - `file git lsof strace`
    - `pciutils usbutils`
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
- to find orphaned files,
  - `cat /var/lib/dpkg/info/*.list | sed -e 's,^/\(bin\|lib\|sbin\),/usr/\1,' | sort | uniq > dpkg.list`
  - `find / -xdev -path /home/* -prune -o -print | sort > find.list`

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

- `gcc-11` provides the C compiler
  - mainly `/usr/bin/x86_64-linux-gnu-gcc-{,ar-,nm-,ranlib-}11`
    - as well as symlinks, `/usr/bin/gcc-{,ar-,nm-,ranlib-}11`
  - it depends on several other packages
  - `cpp-11` provides the C preprocessor
    - `/usr/bin/x86_64-linux-gnu-cpp-11`
  - `binutils` provides the binary utils
    - mainly symlinks such as `/usr/bin/{ld,ld.bfd,ld.gold,nm,readelf,...}`
    - its dependency `binutils-x86-64-linux-gnu` provides the real binaries
      - mainly `/usr/bin/x86_64-linux-gnu-*`
  - `libgcc-11-dev` provides the runtime and intrinsics headers
    - `/usr/lib/gcc/x86_64-linux-gnu/11/{crtbegin.o,libgcc.a,libasan.*,libatomic.*}`
    - `/usr/lib/gcc/x86_64-linux-gnu/11/include/{x86intrin.h,...}`
  - `libc6` provides the C runtime
    - `/lib/x86_64-linux-gnu/{ld,libc,libm,libpthread,...}-2.33.so`
  - it also recommends `libc6-dev`
- `g++-11` provides the C++ compiler
  - mainly `/usr/bin/x86_64-linux-gnu-g++-11`
  - it depends on several other packages
  - `libstdc++6` provides C++ runtime
    - `/usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.29`
  - `libstdc++-11-dev`
- aarch64 cross-compiler
  - `gcc-11-aarch64-linux-gnu` provides the C compiler
    - mainly `/usr/bin/aarch64-linux-gnu-gcc-{,ar-,nm-,ranlib-}11`
    - `cpp-11-aarch64-linux-gnu` provides the C preprocessor
    - `binutils-aarch64-linux-gnu` provides the binary utils
      - mainly `/usr/bin/aarch64-linux-gnu-*`
    - `libgcc-11-dev-arm64-cross` provides the runtime
      - mainly `/usr/lib/gcc-cross/aarch64-linux-gnu/11/*`
    - `libc6-arm64-cross` provides C runtime
      - mainly `/usr/aarch64-linux-gnu/lib/*`
  - `g++-11-aarch64-linux-gnu` provides the C compiler
    - mainly `/usr/bin/aarch64-linux-gnu-g++-11`
    - `libstdc++-11-dev-arm64-cross`
      - `/usr/aarch64-linux-gnu/*`
- i686 cross-compiler
  - `gcc-11-i686-linux-gnu` provides the C compiler
    - it works the same way as the aarch64 cross compiler
  - `gcc-11-multilib-i686-linux-gnu` depends on the C compiler and other
    libraries... for convenience?
  - this conflicts with `gcc-multilib`... which is another way to
    cross-compile for i686?
