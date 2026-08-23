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
  - `sed -i 's/^sudo/#sudo/' install_steam.sh`
  - `./install_steam.sh`
- `BOX64_LOG=1`
- box64 does not seem to be stable to run steam
  - there also appears to be no vk driver when used with umu

## Steam

- <https://wiki.fex-emu.com/index.php/Steam>

## Linux ARM64

- when compiled and run as `FEX`, it acts as a usermode emulator
  - `FEX` is a native arm64 binary that provides CPU and OS simulation
- CPU simulation recompiles x86-64 instrs to arm64 instrs
- OS simulation provides a x86-64 linux kernel
  - elf loading and env setup
  - redirect fs access to a pre-existing x86-64 rootfs
  - synthesize `/proc/cpuinfo`, etc.
  - simulate syscalls
  - simulate signals
  - simulate tls
  - etc.
- Library Thunks
  - for selected libraries, provide x86-64 thunks that forward calls to real
    arm64 libraries
    - sdl, vulkan, gl, x11, wayland, asound, etc.
  - how it works
    - x86-64 app calls into x86-64 thunk
    - x86-64 thunk packs the args and traps to fex
    - fex handles the trap and calls into aarch64 thunk
    - aarch64 thunk unpacks the args and calls the real aarch64 lib

## Windows ARM64

- when compiled as `libarm64ecfex.dll` for x86-64 or `libwow64fex.dll` for
  x86-32, they act as CPU simulators
- they only recompile x86-64 and x86-32 instrs to arm64 instrs
- wine 10+ or win11+ provides the rest
- ARM64EC is similar to library thunks
  - a dll is compiled for ARM64EC rather than for ARM64
    - such a dll is fat and can be loaded as ARM64 or X64
    - compiler generates X64 fast forward stubs
  - how it works
    - X64 app calls into X64 fast forward stub
    - the stub traps to the fex
    - fex calls the real ARM64 version in the same dll
- if a dll is not compiled for ARM64EC but only for ARM64, FEX needs to jit
  everything
- in an ideal setup,
  - wine provides arm64ec (and x86-32) dlls
  - dxvk and vkd3d-proton provide arm64ec (and x86-32) dlls
  - fex provides `libarm64ecfex.dll` (and `libwow64fex.dll`)
  - arm64, x86-64 (and x86-32) apps will work
  - and wine is pure arm64 to linux
