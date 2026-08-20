FEX
===

## Installation

- build
  - `pacman -S clang lld`
  - `pip install packaging`
  - `git clone --recurse-submodules https://github.com/FEX-Emu/FEX.git`
  - `cmake -S . -B out -G Ninja \
       -DCMAKE_INSTALL_PREFIX=$PWD/out/install -DCMAKE_BUILD_TYPE=Release \
       -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
       -DUSE_LINKER=lld -DENABLE_LTO=True \
       -DBUILD_FEXCONFIG=False -DBUILD_TESTING=False -DENABLE_ASSERTIONS=False`
  - `ninja -C out install`
- binfmt
  - `sudo cp out/install/lib/binfmt.d/FEX-x86* /etc/binfmt.d`
  - `sudo systemctl restart systemd-binfmt`
    - `ls /proc/sys/fs/binfmt_misc` to confirm
- x86 rootfs
  - `pacman -S erofsfuse erofs-utils`
  - `FEXRootFSFetcher` downloads and unpacks rootfs to
    `~/.local/share/fex-emu/RootFS`
    - make sure `~/.config/fex-emu/Config.json` point to the rootfs
- run
  - `FEXBash` execs `FEX /bin/bash`, the bash in x86 rootfs
    - `FEX` seems to weird things in addition to translate x86 to arm64
    - e.g., `ls /usr/bin` lists x86 rootfs while `cd /usr/bin && ls` lists
      host rootfs
- optionally customize prebuilt rootfs
  - `pacman -S patchelf`
  - `cd ~/.local/share/fex-emu/RootFS/<foo>`
  - `./chroot.py chroot`
    - `DoUnbreak` preps chroot
      - copy `FEX`, `FEXServer`, and deps to `<foo>/fex`
      - patch copied binaries to modify `PT_INTERP` and add `DT_RUNPATH`
        - it is invalid to patch `/usr/lib/ld-linux-aarch64.so.1`!
      - restore some dirs from `<foo>/chroot` to `<foo>`
      - mount `/proc`, `/sys`, `/dev`, `/dev/pts`, and `/tmp`
    - `sudo chroot <foo> /fex/bin/FEX /usr/bin/bash -i`
    - `DoBreak` cleans up chroot
      - remove `<foo>/fex`
      - unmount pseudo fs
      - move some dirs from `<foo>` to `<foo>/chroot` for backup
- custom rootfs
  - `sudo debootstrap --arch amd64 --variant minbase trixie trixie`
  - `echo "http://deb.debian.org/debian" | sudo tee trixie/debootstrap/mirror`
  - copy fex
    - `sudo cp FEX FEXServer trixie/usr/lib`
    - `sudo cp /usr/lib/{ld-linux-aarch64.so.1,libfmt.so.12,libxxhash.so.0,libstdc++.so.6,libm.so.6,libgcc_s.so.1,libc.so.6} trixie/usr/lib`
  - mount pseudo fs
    - `sudo mount -t proc none trixie/proc`
    - `sudo mount -t sysfs none trixie/sys`
    - `sudo mount -t devtmpfs none trixie/dev`
    - `sudo mount -t devpts none trixie/dev/pts`
    - `sudo mount -o bind /tmp trixie/tmp`
  - `sudo chroot trixie /usr/lib/FEX /debootstrap/debootstrap --second-stage`
  - `sudo chroot trixie /usr/lib/FEX /usr/bin/bash -i`

## Box64

- build
  - `git clone https://github.com/ptitSeb/box64`
  - `cmake -S . -B out -G Ninja \
       -DCMAKE_INSTALL_PREFIX=$PWD/out/install -DCMAKE_BUILD_TYPE=Release \
       -DBOX32=ON -DBOX32_BINFMT=ON \
       -DUSE_CCACHE=1 \
       -DSDORYON1=ON`
- install wrapped x86 libraries
  - `sudo cp -rd x64lib /usr/lib/box64-x86_64-linux-gnu`
  - `sudo cp -rd x86lib /usr/lib/box64-i386-linux-gnu`
- install bundled x86 libraries
  - `sudo pacman -S rpm-tools`
  - `./box64-bundle-x86-libs.sh`
  - `sudo tar -xf box64-bundle-x86-libs.tar.gz --no-same-owner -C /`
- install binfmt
  - `sudo cp out/system/box*.conf /etc/binfmt.d`
- install steam
  - `./install_steam.sh`
- `BOX64_LOG=1`

## UMU Launcher

- <https://github.com/Open-Wine-Components/umu-launcher>
  - a launcher that minics steam to download and use proton to run win games
  - it downloads `UMU-Proton` to `~/.local/share/Steam/compatibilitytools.d`
    - this is to run win games
  - it downloads steamrt to `~/.local/share/umu`
    - this is to run proton
  - it created wineprefix at `~/Games/umu/umu-default`
- build
  - `pacman -S scdoc`
  - `pip install build hatchling installer`
  - `./configure.sh --prefix=$PWD/out/install --use-system-pyzstd --use-system-urllib`
  - `make install`
- run
  - `pip install pyzstd urllib3 xlib`
  - `umu-run <game.exe>`
- debug
  - `UMU_LOG=1`
  - `PROTON_LOG=1`

## Steam

- <https://wiki.fex-emu.com/index.php/Steam>
