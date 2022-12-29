GNU Toolchain
=============

## overview

- binutils: as, ld, readelf, ...
- libc: libc.so, libpthread.so, ld-linux.so, ldconfig, iconv, ...
- gcc: gcc, libgcc.a, ... (runtime might have multiple versions, called multilib)
- `libstdc++`: it is a part of the gcc.  An C++ app built with gcc x.y.z needs
  `libstdc++` x.y.z to run.

## gcc

- `gcc` is a frontend.  It searches for three things
  - toolchain programs, like `cc1`, `as` or `ld`
  - startup files, like `crt?.o` or `libgcc.a`
  - include files
- the former two are listed as programs and libraries in `-print-search-dirs`
  the search paths include hardcoded ones and runtime relative (to gcc binary) ones
- include files are searched in this order
  - in local include dir (`/usr/local/include`, changable at configure time by
    `--with-local-prefix`)
  - prefix include dir (`${prefix}/include`)
  - gcc's include dir (`${libdir}/gcc/<arch>/<version>/include`) 
  - standard include dir (`/usr/include`)
  - standard include dir might not set or might be prefixed by sys-root
- sysroot prefixes system dirs (e.g. `/usr/include` or `/usr/lib`) so that
  libraries/includes from host system will no be used.
- `-isystem` puts a dir before the default ones (but after `-I`)
  - they are treated specially in that some warnings (unused variable, etc.)
    are disabled
- `gcc -v -o b b.o`
- when configured `--disable-shared`, gcc/gcc/ is compiled without
  `ENABLE_SHARED_LIBGCC` defined which modifies some ld rules (to not use
  libgcc, etc.).  `gcc/libgcc/` compiles `libgcc_eh`
- `inhibit_libc`: when configuring cross compiler without `--with-headers`, if
  `--with-newlib` or no `--with-sysroot` is given, libc is inhibited

## gcc specs

- `gcc -dumpspecs` to show built-in specs.
- `gcc -specs=<file>` to specify a specs file.
- a specs file consists of spec strings.  Each is started by a directive, and is
  seperated by blank lines.  There are three types of directives
  1. `%COMMAND`: Only `%include`, `%include_noerr`, and `%rename` are allowed
  2. `*[SPEC_NAME]`: Create/Override/Delete the named spec string
  3. `[SUFFIX]`: How to compile a file with the given suffix
- `*[SPEC_NAME]` usually overrides an existing spec string, unless
  - the new spec string begins with a `+` and the new one appends to old one
  - the old one is `%rename`ed and the new one inserts it through `%(old_one)`
- some of the built-in spec strings

       asm          Options to pass to the assembler
       asm_final    Options to pass to the assembler post-processor
       cpp          Options to pass to the C preprocessor
       cc1          Options to pass to the C compiler
       cc1plus      Options to pass to the C++ compiler
       endfile      Object files to include at the end of the link
       link         Options to pass to the linker
       lib          Libraries to include on the command line to the linker
       libgcc       Decides which GCC support library to pass to the linker
       linker       Sets the name of the linker
       predefines   Defines to be passed to the C preprocessor
       signed_char  Defines to pass to CPP to say whether `char' is signed
                    by default
       startfile    Object files to include at the start of the link
- e.g., there is a built-in suffix directive saying that `.c` is of language C
  (`@c`).  And another rule says that inputs of language C are compiled with
  `cc1 %(cpp_unique_options) %(cc1_options)`, where `%(cpp_unique_options)`
  usually has `%C` expanding to `%(cpp)` and `%(cc1_options)` has `%1` expanding
  to `%(cc1)`.
  - `.h` is mapped to `@c-header`.  It is compiled with
  `cc1 %(cpp_unique_options) %(cc1_options)`.
- e.g., `*link_command` has, among others, in this order
  - `%l`: for `%(link)`
  - `%X`: for all `-Wl,blah`
  - `%{o*}`: for `-o blah`
  - `%S`: for startfile
  - `%{L*}`: all `-Ldir`
  - `%(link_libgcc)`: usually `%D`, to add all directories which might have
    startfile to search list.
  - `%o`: all `.o` and `-lfoo`
  - `%(link_gcc_c_sequence)`: usually `%G %L %G`, for `%(libgcc) %(lib) %(libgcc)`
  - `%E`: for endfile
- aboult `-lfoo`

    GCC also knows implicitly that arguments starting in `-l` are to be
    treated as compiler output files, and passed to the linker in their
    proper position among the other output files.

## libgcc

- internal routines (e.g. for float number support) used in compiled binaries
- see `install-leaf` and `install-shared` (if shared) for installed binaries
- its Makefile includes `$(gcc_objdir)/libgcc.mvars`
- `libgcc_eh.a`: exception handler
  - Usually, `LIB2ADDEH == LIB2ADDEHSTATIC == LIB2ADDEHSHARED`
  - When configured non-shared, it is added to `libgcc.a`
- in config.host, `extra_parts="crtbegin.o crtbeginS.o crtbeginT.o crtend.o crtendS.o"`
  for arm

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

## crtX

- removing `/usr/lib/crt*.o` provided by glibc makes gcc fail to compile executable
- removing `/usr/lib/gcc/<target>/<version>/crt*.o` provided by gcc itself makes
  gcc fail to compile executable
- removing `/usr/lib/gcc/<target>/<version>/libgcc.a` provided by gcc itself makes
  gcc fail to compile executable
- when compiling gcc, a list of objects (crtX) to link when compiling is hardcoded.
- in a compiled x86 shared executable, there are:

        w __gmon_start__
        T __libc_csu_fini
        T __libc_csu_init
        U __libc_start_main@@GLIBC_2.0
        T _fini
        T _init
        T _start
        T main
  - `crti.o`: initializer `.init`, built from `initfini.c`
    - defines `_init` and `_fini`
  - `crt1.o`: startup code, at the beginning of `.text`, built from `start.[cS]`
    - defines `_start`, calls to `__libc_csu_init` and `__libc_start_main`
  - `crtn.o`: finalizer `.fini`, built from `initfini.c`
  - `crtbegin.o`, `crtend.o`: from gcc, appended to `.init` and `.fini`

## lifecycle (x86)

- ld.so jumps to `_start` function defined in `sysdeps/i386/elf/start.S` (`crt1.o`)
- `_start` pushes `__libc_csu_init`, `__libc_csu_fini`, `main`, `argc`, `argv`, etc.
  to stack and calls `__libc_start_main` defined in `csu/libc-start.c` (`libc.so`)
- `__libc_start_main` calls, among others, `__libc_csu_init` (and then main) defined in
  `csu/elf-init.c` (`libc_nonshared.a`)
- `__libc_csu_init` calls `_init`, defined in (generated) `csu/initfini.c` (`crti.o`)

## glibc

- see `configure.in` and `sysdeps/unix/sysv/linux/configure.in`
- usually, `libdir=$(prefix)/lib` and `inst_libdir=$(install_root)$(libdir)`
  - `slibdir=/lib` and `inst_slibdir=$(install_root)$(slibdir)`
- for `libc.so`, a ld script is installed to `inst_libdir` and the real `.so`
  is installed to `inst_slibdir`

## OS ABI

- ELF has a section `.note.ABI-tag`.
- When ldconfig runs, the same library with higher OS version will be sorted
  first.
- It causes the library to be used, like the `libGL.so` on my debian unstable.

## GCC Visibility

- <http://gcc.gnu.org/wiki/Visibility>
- `default` means public, `hidden` means private
- `-fvisibility=hidden` means default to hidden
- A common practice is to compile with `-fvisibility=hidden` and
  use `__attribute__(visibility("default"))` to punch holes.
- Shared libraries are PIC and their functions are called through PLT, even for
  non-API global functions.  These functions should be marked as hidden to avoid
  PLT.
