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
