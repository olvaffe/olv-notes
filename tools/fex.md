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
- rootfs
  - `pacman -S erofsfuse erofs-utils patchelf`
  - `FEXRootFSFetcher` downloads rootfs
    - unpack manuall as root
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
