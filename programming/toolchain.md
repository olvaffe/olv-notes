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

## PIC (Position Independent Code)

- <http://www.greyhat.ch/lab/downloads/pic.html>
- For functions, relative jumps are used
- For static/global variables or strings, GOT entries are used
  - The GOT entries are derived from `__i686.get_pc_thunk.bx` and an
    offset.  In C, it is `GOT[offset]`, which is the absolute address of the
    variable in current process.  The offset is generated by linker through
    `R_386_GOTPC` to find GOT and `R_386_GOTOFF` to find entry.
  - The same trick can be used locate the variable directly.  Why go through
    GOT?
    - Consider our PIC code is linked to a shared library and the variables it
      access are in that library.  Since the addresses of the variables are
      unknown at link time, we cannot derive the needed offsets to the
      variables.  Therefore, we need GOT.  GOT can be modified by dynamic
      loader.
    - Probably because we do not want to (and cannot in the case of linked to a
      shared library) force text and data sections to have constant offset.
      With GOT, it is GOT that has constant offset from text section.  Data
      section may be put in a better place to share with other processes.
- Now go back to functions again.  What if functions are in some libraries?
  What if we want functions to be overloaded?
  - We cannot, in the former case, and may not want to, in the latter case, call
    the function directly using relative addresses.
  - The soluction is PLT.  In both cases, instead of calling `func`, we call
    `func@plt`.  The first time `func@plt` is called, it calls into dynamic
    loader to locate and store the address of `func` in GOT (`.got.plt`).  Later
    on, `func@plt` can jump to `func` directly.  This also has the benefit of
    lazy binding.
  - By default, global functions of our code are overloadable.  That is, there
    are PLT entries for global functions and they are used.  Static functions
    are not overloadable, no PLT entries for them.  To make global functions not
    overloadable, specify `-Bsymbolic` in ld.
- An executable is usually not PIC.  It can access its functions/variables using
  addresses decided at compile time.
  - How does it access library's variables?  It emits `R_386_COPY` so that
    library's variables are copied to its `.bss`.  It makes them variables _in_
    the executable.
  - How about library's functions?  Just like PLT, with GOT is a known place.
- To call a PIC function `foo`,
  - the caller calls `foo@plt` in section `.plt`
  - `foo@plt` jumps to suitable entry in `.got.plt` directly
  - The entry in `.got.plt` jumps back to `foo@plt+6`, which pushes entry number
    and jumps to the zero-th entry of `.plt`.
  - The zero-th entry of `.plt` is special and it calls dynamic linker to
    resolve the symbol and stores it in `.got.plt`.
  - The next time `foo@plt` is called, it jumps twice to `foo`.
- Here is a sample code

    #include <stdio.h>
    
    static void a(void) { }
    void b(void) { }
    
    int main(void)
    {
    	printf("hello world\n");
    	a();
    	b();
    	return 0;
    }
  - Compile it normally or with `-fPIC`, `-Wl,-Bsymbolic`, etc.  See its
   assembly output!

## PIC References

- <http://www.airs.com/blog/archives/41>

## PIC Old


GOT for static or global variables access

1. Get the offset of GOT

    1d9c:       e8 b6 fb ff ff          call   1957 <__cxa_finalize@plt+0xb3>
    1da1:       81 c3 67 28 01 00       add    $0x12867,%ebx   # the magic number is generated by asm("add _GLOBAL_OFFSET_TABLE_ - ., %ebx")
 
where 1957 gives

    1957:       8b 1c 24                mov    (%esp),%ebx
    195a:       c3                      ret

2. access variables through GOT


GOT for global functions

0. background

Call to global functions becomes
  51:   e8 fc ff ff ff          call   52 <main+0x33>
                        52: R_386_PLT32 strerror

instead of
  38:   e8 fc ff ff ff          call   39 <main+0x27>
                        39: R_386_PC32  strerrror

which will create a PLT entry and call to it.  A relocation to GOT is created,
which is lazy.

000015c4 <sprintf@plt>:
    15c4:       ff a3 10 00 00 00       jmp    *0x10(%ebx)
    15ca:       68 08 00 00 00          push   $0x8
    15cf:       e9 d0 ff ff ff          jmp    15a4 <ZLIB_1.2.0+0x15a4>

1. respective %ebx + 0x10 initializes (lazy!) to 15ca + (library real address - library load address in file)
2. the first time sprintf is called, jump into 15a4, first entry in .plt

000015a4 <__errno_location@plt-0x10>:
    15a4:       ff b3 04 00 00 00       pushl  0x4(%ebx)
    15aa:       ff a3 08 00 00 00       jmp    *0x8(%ebx)

3. %ebx+0x8 points to some function in ld.so, which finds the real address of sprintf (use the relocation) and fill in (%ebx+0x10)
