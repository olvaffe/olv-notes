Cross Toolchains
================

## Debian

- gcc toolchains
  - `sudo apt install {gcc,g++}-aarch64-linux-gnu`
    - this installs the cross toolchains to `/usr` like normal packages
    - they depend on cross C/C++ runtimes which get installed to
      `/usr/aarch64-linux-gnu` to form a mini sysroot
    - they are configured to search in the mini sysroot (for C/C++ runtimes)
  - `aarch64-linux-gnu-gcc --sysroot=<path> ...`
    - `--sysroot=<path>` is the correct syntax and `--sysroot <path>` is wrong
- llvm toolchains
  - `sudo apt install clang`
    - clang supports multiple targets and does not need separate packages
    - but cross C/C++ runtimes are provided by separate packages
  - `clang -target aarch64-linux-gnu --sysroot=<path> ...`
- rust
  - `rustup target add aarch64-unknown-linux-gnu`
    - rustc is LLVM-based and supports multiple targets
    - this installs cross rust runtimes
    - but linkers still need to configured separately in `~/.cargo/config.toml`
      - `[target.aarch64-unknown-linux-gnu]`
      - `linker = "aarch64-linux-gnu-gcc"`
  - `cargo build --target aarch64-unknown-linux-gnu`
- sysroot
  - `sudo apt install debootstrap qemu-user-static`
  - `sudo debootstrap --arch arm64 testing my-sysroot`
  - `sudo chroot my-sysroot`
  - note that if the sysroot uses a newer glibc than the host cross toolchains
    does, libraries in the sysroot may reference symbols that the host cross
    glibc does not provide
    - make sure this does not happen!

## Android NDK

- llvm toolchains
  - download NDK and they are under `$NDK/toolchains/llvm/prebuilt/linux-x86_64`
  - executables are under `bin/`
  - mini sysroots are under `sysroot/`
- rust
  - configure to use the linker from the llvm toolchains
- sysroot
  - build from scratch

## Chrome OS SDK

- llvm toolchains
  - <https://storage.googleapis.com/chromiumos-sdk/2024/08/x86_64-cros-linux-gnu-2024.08.07.69651.tar.xz>
  - executables are under `bin/`
  - mini sysroot is under `./`
- rust
  - configure to use the linker from the llvm toolchains
- sysroot
  - build from scratch
  - or, point to `$CHROMIUMOS/out/build/$BOARD/`

## Arch Linux Cross Toolchains

- `aarch64-linux-gnu-gcc` is a cross toolchain
  - it depends on `aarch64-linux-gnu-binutils`
  - it also depends on `aarch64-linux-gnu-glibc`
    - which in turn depends on `aarch64-linux-gnu-linux-api-headers`
- contents of `aarch64-linux-gnu-binutils`
  - executables (nm, ld, strip, etc.) are installed to `/usr/bin`
    - they are prefixed
    - unprefixed ones are installed to `/usr/aarch64-linux-gnu/bin`
      - they should not be used
      - they are for old projects who hard code linker, etc.
  - linker scripts are installed to `/usr/aarch64-linux-gnu/lib/ldscripts`
- contents of `aarch64-linux-gnu-linux-api-headers`
  - kernel uapi is installed to `/usr/aarch64-linux-gnu/include`
- contents of `aarch64-linux-gnu-glibc`
  - headers are installed to `/usr/aarch64-linux-gnu/include`
  - runtime and helpers are installed to `/usr/aarch64-linux-gnu/lib`
    - `libc`, `libm`, `libdl`, `libpthread`, `libresolv`, etc.
    - `ld-linux-aarch64`, `crt1.o`, etc.
- `aarch64-linux-gnu-gcc`
  - executables are installed to `/usr/bin`
  - helpers are installed to `/usr/lib/gcc/<arch>/<version>`
    - `cc1`, `crtbegin.o`, `crtend.o`, etc.
    - `stddef.h`, `stdbool.h`, etc.
      - yeah, some std headers are in gcc
  - runtime is installed to `/usr/aarch64-linux-gnu/lib64`
    - `libgcc_s`, `libgomp`, etc.
  - `libstdc++` is a part of g++
    - headers are installed to `/usr/aarch64-linux-gnu/include`
    - libraries are installed to `/usr/aarch64-linux-gnu/lib64`
- sysroot
  - the gcc cross toolchain uses `/usr/aarch64-linux-gnu` as the sysroot
    - `aarch64-linux-gnu-gcc -print-sysroot`
  - if needing anything beyond glibc/stdc++, bootstrap arch linux arm as the
    sysroot instead
    - `pacman -S qemu-user-static qemu-user-static-binfmt`
    - `systemctl start systemd-binfmt`
    - `wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz`
    - not working
      - `aarch64-linux-gnu-gcc -print-search-dirs` confirms the search dirs
         are correct but still not working
      - `mv /usr/aarch64-linux-gnu/lib/libc.so /tmp` allows it to work
        - that file is a ld script and hard codes `/lib/libc.so.6`
        - this issue is arch-specific.  On debian,
          `/usr/aarch64-linux-gnu/lib/libc.so` is installed by
          `libc6-dev-arm64-cross` and hard codes
          `/usr/aarch64-linux-gnu/lib/libc.so.6` instead
- clang
  - clang can cross-compile to many targets by default
  - need `-target aarch64-linux-gnu --sysroot=/usr/aarch64-linux-gnu`

## crostool-ng

- build
  - `./bootstrap`
  - `./configure --enable-local`
  - `./ct-ng menuconfig` and configure the toolchain
  - `./ct-ng build.128`
  - the cross toolchain is at `$HOME/x-tools`
- internally, it builds these packages, in order
  - `zlib`, `gmp`, `mpfr`, `isl`, `mpc`, `ncurses`, `libiconv`, `gettext`
  - `binutils`, kernel headers, core `gcc`, `glibc`, final `gcc`
- sysroot
  - <https://crosstool-ng.github.io/docs/toolchain-usage/> documents 4 ways to
    use a sysroot
- internal
  - `prefix=~/x-tool/<arch>`
  - `sysroot=~/x-tool/<arch>/<arch>/sys-root`
  - sys-root contains:
    - `/lib/ldscripts/*` (scripts from binutils)
    - `/usr/include/*` (kernel and glibc headers)
  - sysroot is used by gcc and ld for seaching libraries and includes (see
    manpages)
  - gcc `--with-local-prefix` is set to sys-root to specify the local dir
    (default to `/usr/local`)

## buildroot

- build
  - `make menuconfig` and configure the toolchain
  - `make -j128 toolchain`
  - the cross toolchain is at `output/host/bin`
- internally, it builds these packages, in order
  - `m4`, `bison`, `gawk`, `binutils`, `gmp`, `mpfr`, `mpc`
  - `gcc-initial`, `linux-headers`, `glibc`, `gcc-final`

## overview

- <http://www.gentoo.org/proj/en/base/embedded/handbook/?part=1&chap=4#doc_chap3>
- `gcc -v -o a a.c` to see the real commands issued
- sysroot: THE ONLY RIGHT THING
- glibc: some parts need `libgcc.a`, some needs `libgcc_eh.a` (need shared gcc)
- shared gcc: needs `libc.so`, `crt?.o`, libc headers, because it might insert code
  for pointer checking, profiling, etc. and needs libc.

## system type

- CPU-VENDOR-OS tuple, where OS can be SYSTEM or KERNEL-SYSTEM
  - e.g. i686-pc-linux-gnu
  - `--build` gives the type of the building system
  - `--host` gives the type the app will run
  - `--target` gives the type of system the app will deal with
- when `--host` is given, `host_cpu`, `host_vendor`, and `host_os` is set
- there are convensions.  type of `arm*-*-linux*` or `arm*-*-linux-*eabi`
  like `arm-unknown-linux-gnueabi` or `arm-unknown-linux-uclibcgnueabi` should
  be used.
- `--host` enables cross-compiling: use toolchain with the given prefix
- see also crossgcc ml

## SYSROOT

- please use it!
