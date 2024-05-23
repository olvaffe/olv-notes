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
- link manually
  - `gcc -c hello.c`
  - `ld -pie --dynamic-linker /lib64/ld-linux-x86-64.so.2 -o hello /usr/lib/crt1.o hello.o -lc`
- interesting options
  - `--entry=NAME` specifies the entrypoint
    - default to `_start`, or the beginning of `.text` section
  - `--export-dynamic` adds all symbols to `.dynsym`
    - when the exec dlopens, this allows the opened library to refer back to
      symbols defined in the executable
  - `-init=NAME` specifies `DT_INIT` function, which is invoked on load
    - default to `_init`
  - `-fini=NAME` specifies `DT_INIT` function, which is invoked on unload
    - default to `_fini`
  - `-soname=name` specifies `DT_SONAME`
    - when an exec links to a shared lib with `DT_SONAME`, its `DT_NEEDED`
      will use `DT_SONAME` rather than the real name of the shared lib
  - `-lNAME` specifies a file to link
    - ld searches `libNAME.so` before `libNAME.a`
    - in the case with `libNAME.o`, ld adds `DT_NEEDED`
    - in the case with `libNAME.a`, ld adds the necessary objects from
      the archive
  - `-LDIR` specifies a dir to search for libraries
  - `-o NAME` specifies the output file
  - `--relocatable` creates a `ET_REL`
  - `--script=scriptfile` specifies an alternative link script rather than the
    default link script
  - `--undefined NAME` adds NAME as an undefined symbol
    - this can be used to pull in more objects from `.a` archive
  - `-z keyword`
    - `pack-relative-relocs` generates compact relative relocation for PIC
      libraries and executables.  It adds both `DT_RELR` and
      `GLIBC_ABI_DT_RELR`
      - other linkers support `--pack-dyn-relocs=relr` which does not add
        `GLIBC_ABI_DT_RELR`.  glibc might refuse to load.
      - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/sys-devel/binutils/files/ldwrapper_lld.hardened>
        - cros uses `--pack-dyn-relocs=relr`
  - `--start-group archives --end-group` searches archives repeatedly unless
    there is no new undefined symbol
    - this is useful when archives have mutual dependencies
  - `--as-needed` causes `DT_NEEDED` to be emitted only when the shared
    library provides an otherwise undefined symbol
  - `-Bdynamic` makes `-lNAME` search `libNAME.so` in addition to `libNAME.a`
    - this is the default
  - `-Bstatic` or -`static` makes `-lNAME` search only `libNAME.a`
  - `-Bsymbolic` emits `DT_SYMBOLIC` to the shared lib
    - when the shared lib refers a global symbol that is also defined by the
      lib itself, the dynamic linker will resolve the symbol to that
      definition
    - this breaks `LD_PRELOAD`
  - `-Bsymbolic-functions` disables relocations for global functions defined
    by the shared lib itself
    - those global functions are resolved to their local definitions
    - this breaks `LD_PRELOAD`
  - `--dynamic-list=FILE` specifies a file which contains a list of symbols
    - for shared lib, the listed symbols will not be bound to local
      definitions
    - for exec, the listed symbols will be added to `.dynsym`
  - `--dynamic-linker=LINKER` specifies the dynamic loader
    - `/lib64/ld-linux-x86-64.so.2` nowadays
  - `--no-undefined` disallows undefined symbols in shared lib
    - by default, a shared lib can have undefined symbols because the exec
      loading the shared lib may provide the symbols
  - `-nostdlib` ignores search dirs specified in the link script
  - `-pie` creates a `ET_DYN` exec rather than a `ET_EXEC` exec
  - `-rpath=DIR` emits `DT_RUNPATH` to exec
    - `DT_RUNPATH` is additional shared lib search dirs used by the dynamic
      loader
    - older ld emits `DT_RPATH` instead, which breaks `LD_LIBRARY_PATH` and is
      very bad
  - `-rpath-link=DIR` is the search dir for indirect shared libs
    - when an exec links to libFOO, and libFOO depends on libBAR, this tells
      ld where to find libBAR
    - this is for ld link time and does not affect the dynamic loader
  - `-shared` creates a `ET_DYN` shared lib
  - `--sysroot=SYSROOT` specifies a sysroot
  - `--version-script=SCRIPT` specifies a version script
  - `--build-id` emits `.note.gnu.build-id` section
- version scripts
  - it is meaningful only when creating shared libs
  - `{ global: foo; local: *; }` means all symbols are bound to an unspecified
    base version
    - only `foo` is public

## binutils

- GNU EABI == no eabi, EABI4 == EABI5 == eabi
