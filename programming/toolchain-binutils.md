binutils
========

## overview

- binutils: as, ld, readelf, ...
- libc: libc.so, libpthread.so, ld-linux.so, ldconfig, iconv, ...
- gcc: gcc, libgcc.a, ... (runtime might have multiple versions, called multilib)
- `libstdc++`: it is a part of the gcc.  An C++ app built with gcc x.y.z needs
  `libstdc++` x.y.z to run.

## ld

- ld searches for libraries.  It uses sysroot too.  If sysroot given
  at configuration time is a decendant of prefix dir, the sysroot is
  relocatable, according to where the current executables are, which
  is strongly recommended.
- when ld runs, `get_sysroot` is always called.  It might be empty if
  not compiled with sysroot support.  `${prefix}/bin` is called `BINDIR`,
  `${prefix}/<target>/bin` is called `TOOLBINDIR`
- default search pathes are given by `ldscripts`.  They are usually
  absolute, prefixed by sysroot if supported.  ld only needs to find
  ldscripts and the rest is given by script or by command line
- `-L` describes the linker search path
- `-rpath` describes the dynamic linker search path
- `-rpath-link` describes the search path the linker should use when it is
  emulating dynamic linker search path
- prog links to libA, which links to libB.  At linking time, libB is
  automatically added to prog, and linker emulates dynamic linker for that
  purpose.  If rpath is used instead of rpath-link, the path leaks into prog.
  - It could be dangerous and overrides `LD_LIBRARY_PATH`

## binutils

- GNU EABI == no eabi, EABI4 == EABI5 == eabi
