Cross Toolchain
===============

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
  - need `-target aarch64-linux-gnu --sysroot /usr/aarch64-linux-gnu`

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
- glibc: some parts need libgcc.a, some needs libgcc_eh.a (need shared gcc)
- shared gcc: needs libc.so, crt?.o, libc headers, because it might insert code
  for pointer checking, profiling, etc. and needs libc.

## crosstool-ng

- prefix=~/x-tool/<arch>
- sysroot=~/x-tool/<arch>/<arch>/sys-root
- sys-root contains:
  /lib/ldscripts/* (scripts from binutils)
  /usr/include/* (kernel and glibc headers)
- sysroot is used by gcc and ld for seaching libraries and includes (see
  manpages)
- gcc --with-local-prefix is set to sys-root to specify the local dir (default
  to /usr/local)

## system type

- CPU-VENDOR-OS tuple, where OS can be SYSTEM or KERNEL-SYSTEM
- e.g. i686-pc-linux-gnu
- --build gives the type of the building system
- --host gives the type the app will run
- --target gives the type of system the app will deal with
- when --host is given, host_cpu, host_vendor, and host_os is set
- there are convensions.  type of arm*-*-linux* or arm*-*-linux-*eabi
  like arm-unknown-linux-gnueabi or arm-unknown-linux-uclibcgnueabi should be
  used.
- --host enables cross-compiling: use toolchain with the given prefix
- see also crossgcc ml

## binutils

- ./configure --target=arm-unknown-linux-gnueabi --prefix=/opt/arm --disable-nls
- make
- make install, and it gives
  ${prefix}/arm-unknown-linux-gnueabi/bin/XXX (without prefix)
           /bin/arm-unknown-linux-gnueabi-XXX
           /lib/libiberty.a

## gcc (static)

- a static cross gcc with minimal features to compile libc
- ./configure --target=arm-unknown-linux-gnueabi --prefix=/opt/arm --disable-nls \
  --disable-multilib --disable-threads --enable-languages=c --disable-shared \
  --with-newlib --enable-target-optspace
- make all-gcc, to compile only gcc
- make install-gcc, and it gives
  ${prefix}/arm-unknown-linux-gnueabi/bin/gcc
           /bin/arm-unknown-linux-gnueabi-XXX
           /lib/gcc/arm-unknown-linux-gnueabi/4.5.0/ (only headers and scripts)
           /libexec/gcc/arm-unknown-linux-gnueabi/4.5.0/cc1 and others
- make all-target-libgcc and make install-target-libgcc to install crt* and libgcc.a
- cd arm-unknown-linux-gnueabi/libgcc, make libgcc_eh.a, and make install-shared
  (it fails halfway but that's ok. libgcc_eh.a is dummy when --disable-shared)

## kernel headers

- for use by glibc
- make ARCH=arm headers_install INSTALL_HDR_PATH=/opt/arm/arm-unknown-linux-gnueabi/usr

## glibc

- remember to cd libc and
  cvs -z 9 -d :pserver:anoncvs@sources.redhat.com:/cvs/glibc co ports
- configure --prefix=/usr --host=arm-unknown-linux-gnueabi --without-gd --without-cvs \
  --enable-shared --with-headers=/opt/arm/arm-unknown-linux-gnueabi/usr/include \
  libc_cv_forced_unwind=yes libc_cv_c_cleanup=yes
- note the --prefix is different
- make
- make install install_root=/opt/arm/arm-unknown-linux-gnueabi/ and it gives
  ${install_root}${prefix}/bin: locale, timezone, iconv, etx.
                          /include: libc headers
                          /share: locale infos, etc.
                          /lib: lots of glibc libraries, some are ld scripts
                          more...
  ${install_root}/sbin: ldconfig, etc.
  ${install_root}/lib: real libraries for libc, libpthread, etc.

## gcc

- ./configure --target=arm-unknown-linux-gnueabi --prefix=/opt/arm --disable-nls \
  --disable-multilib --enable-threads --enable-languages=c --enable-shared \
  --enable-target-optspace --with-headers=/opt/arm/arm-unknown-linux-gnueabi/usr/include
- need --with-headers because --with-sysroot is not given
- if binutils is not compiled with sysroot support, copy what's in usr/lib/ to lib/
  before compile (crt?.o, libc.so and libpthread.so scripts with modifications)
- make
- make install and it gives
  ${prefix}/bin/arm-unknown-linux-gnueabi-XXX
           /lib/gcc/arm-unknown-linux-gnueabi/4.5.0: crt*.o, includes, libgcc*.a
           /libexec/gcc/arm-unknown-linux-gnueabi/4.5.0: cc1, collect2
  ${prefix}/arm-unknown-linux-gnueabi/lib: libgcc_s.so, lib{gomp,mudflap,ssp}.so

## SYSROOT

- please use it!
