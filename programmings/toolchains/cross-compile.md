overview
- http://www.gentoo.org/proj/en/base/embedded/handbook/?part=1&chap=4#doc_chap3
- gcc -v -o a a.c to see the real commands issued
- sysroot: THE ONLY RIGHT THING
- glibc: some parts need libgcc.a, some needs libgcc_eh.a (need shared gcc)
- shared gcc: needs libc.so, crt?.o, libc headers, because it might insert code
  for pointer checking, profiling, etc. and needs libc.

crosstool-ng
- prefix=~/x-tool/<arch>
- sysroot=~/x-tool/<arch>/<arch>/sys-root
- sys-root contains:
  /lib/ldscripts/* (scripts from binutils)
  /usr/include/* (kernel and glibc headers)
- sysroot is used by gcc and ld for seaching libraries and includes (see
  manpages)
- gcc --with-local-prefix is set to sys-root to specify the local dir (default
  to /usr/local)

system type
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

binutils
- ./configure --target=arm-unknown-linux-gnueabi --prefix=/opt/arm --disable-nls
- make
- make install, and it gives
  ${prefix}/arm-unknown-linux-gnueabi/bin/XXX (without prefix)
           /bin/arm-unknown-linux-gnueabi-XXX
           /lib/libiberty.a

gcc (static)
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

kernel headers
- for use by glibc
- make ARCH=arm headers_install INSTALL_HDR_PATH=/opt/arm/arm-unknown-linux-gnueabi/usr

glibc
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

gcc
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

SYSROOT
- please use it!

old
- for glibc, ...
- make lib (it fails, but libc.so is built)
- make install install_root=/opt/arm/arm-unknown-linux-gnueabi (it fails,, but libc.so is installed)
- make install-headers install_root=/opt/arm/arm-unknown-linux-gnueabi
- cp bits/stdio_lim.h  /opt/arm/arm-unknown-linux-gnueabi/usr/include/bits/
- mkdir /opt/arm/arm-unknown-linux-gnueabi/usr/lib
- cp csu/crt?.o /opt/arm/arm-unknown-linux-gnueabi/usr/lib/
- glibc: some parts need libgcc.a, some needs libgcc_eh.a
- shared gcc: needs libc.so, crt?.o, libc headers
- configure gcc with --enable-shared and manage to build and install libgcc_eh.a
- glibc can be fully built/installed
- build full gcc
