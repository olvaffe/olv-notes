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
  - `FEXRootFSFetcher` downloads and unpacks rootfs
  - `cd ~/.local/share/fex-emu/RootFS/<foo>`
  - `./chroot.py chroot`
