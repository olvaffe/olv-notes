glibc
=====

## Installation

- <https://sourceware.org/git/glibc.git>
- `libc.so` is a ld script
  - see `configure.in` and `sysdeps/unix/sysv/linux/configure.in`
  - usually, `libdir=$(prefix)/lib` and `inst_libdir=$(install_root)$(libdir)`
    - `slibdir=/lib` and `inst_slibdir=$(install_root)$(slibdir)`
  - for `libc.so`, a ld script is installed to `inst_libdir` and the real
    `.so` is installed to `inst_slibdir`

## `man ld.so`

- a library is searched in the following order
  - if the executable has `DT_RPATH` and does not have `DT_RUNPATH`, search in
    `DT_RPATH`
    - this should be avoided
  - search in `LD_LIBRARY_PATH` if not in secure-execution mode
    - not suid, sgid, etc.
  - if the executable has `DT_RUNPATH`, search in `DT_RUNPATH`
    - only for libraries that are directly `DT_NEEDED` by the executable
    - in contrast, `DT_RPATH` applies to dependencies of the libraries as well
  - search in `/etc/ld.so.cache`
    - it is updated by `ldconfig`
    - `/etc/ld.so.conf` specifies the search directories of `ldconfig`
  - in default paths such as `/lib`, `/usr/lib`, etc.
- string token expansion
  - `$ORIGIN` is expanded to the location of the executable/library
  - `$LIB` is expanded to `lib` or `lib64`
  - `$PLATFORM` is expanded to `x86_64`, etc.
- environment variables
  - `LD_BIND_NOW` resolves all symboles immediately
  - `LD_LIBRARY_PATH` specifies a colon-separated list of search paths
  - `LD_PRELOAD` specifies a colon-seperated list of libraries to preload
  - `LD_DEBUG` prints debug messages
  - more
- cmdline options
  - `/lib64/ld-linux-x86-64.so.2 --list /usr/bin/ls` is like `ldd /usr/bin/ls`
  - `--inhibit-rpath` specifies a colon-seperated of list of objects whose
    `DT_RPATH`/`DT_RUNPATH` should be ignored
    - for the main executable itself, use an empty string

## life cycle

- the elf executable has `ld-linux-<arch>.so` as the interpreter and has
  `_start` as the entry point
  - kernel jumps to the entry point of ld
    - `sysdeps/x86_64/dl-machine.h` defines `RTLD_START` which expands to
      `_start` for ld
    - `_dl_start`
    - `_dl_init`
    - it jumps to the executable entry point
      - `_dl_fini` is a callback called by the executable
  - ld loads the executable and jumps to the executable's entry point
- `sysdeps/x86_64/start.S` defines `ENTRY (_start)`
  - it is a part of `crt1.o`
  - `_start` prepares args and calls `__libc_start_main`
    - arg1 is in `%rdi` and is `main`
    - arg2 is in `%rsi` and is `argc`
      - it is popped from the current stack
    - arg3 is in `%rdx` and is `argv`
      - it points to the current stack top
    - arg4 is in `%rcx` and is `init`
      - it is zeroed
    - arg5 is in `%r8` and is `fini`
      - it is zeroed
    - arg6 is in `%r9` and is `rtld_fini`
      - is is from the ld
    - arg7 is in stack and is `stack_end`
- `csu/libc-start.c` defines `__libc_start_main`
  - it is a versioned symbol and is a part of `libc.so`
  - `LIBC_START_MAIN` macro expands to the the real impl,
    `__libc_start_main_impl`
  - if `rtld_fini` is provided, it is registered to `__cxa_atexit`
  - `call_init` calls init functions
    - elf `DT_INIT` has `_init`
      - it is defined in `sysdeps/x86_64/crti.S`
      - it calls `__gmon_start__`
    - elf `DT_INIT_ARRAY` consists of ctros
      - e.g., `__attribute__ ((constructor))`
      - `csu/init-first.c` defines `_init_first`
  - `__libc_start_call_main` is called
- `sysdeps/nptl/libc_start_call_main.h` defines `__libc_start_call_main`
  - it calls `exit(main(argc, argv))` after doing some inits for threads
- `stdlib/exit.c` defines `exit`
  - `__call_tls_dtors` calls dtors on `tls_dtor_list` list
  - it calls dtors on `__exit_funcs` list
    - these are added by `atexit`
  - `_exit` calls `exit_group` syscall on linux

## Entry Point on x86-64

- for an executable built with `-static`,
  - `readelf` reports
    - `Type: EXEC (Executable file)`
    - `Entry point address: 0x4014e0`
    - a segment in the program header to load a region of the binary to
      `0x0000000000401000` with read-only and executable
  - `objdump` reports
    - the entry point is `_start`
    - `_start` calls `__libc_start_main` with the hard-coded address of `main`
- for an executable built with `-static-pie`,
  - `readelf` reports
    - `Type: DYN (Position-Independent Executable file)`
    - `Entry point address: 0x95a0`
    - a segment in the program header to load a region of the binary to
      `0x0000000000009000` with read-only and executable
  - `objdump` reports
    - the entry point is `_start`
    - `_start` calls `__libc_start_main` with the relative address of `main`
      - `%rip` plus a hard-coded offset, which is the offset from `_start` to
        `main`

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

## `man standards`

- V7, Version 7 UNIX
  - 1979
  - AT&T/Bell Labs
  - then it diverged into BSD and System V dialects
- BSD
  - 4.2BSD (1983), 4.3BSD (1986), and 4.4BSD (1993)
  - there were also older 3BSD (1980), 4BSD (1980), and 4.1BSD (1981)
  - Berkeley
- System V
  - SV (1983), SVr2 (1985), SVr3 (1986), SVr4 (1989)
  - there was also System III (1981)
  - AT&T
- System V Interface Definition
  - SVID 1 formally describes SVr2
  - SVID 2 formally describes SVr3
  - SVID 3 formally describes SVr4
  - SVID 4 (1995)
- POSIX, Portable Operating System Interface
  - POSIX.1-2001 / SUSv3
    - SUS, Single UNIX Specification, requires XSI and adds XCurses
  - POSIX.1-2008 / SUSv4
  - POSIX.1-2017 / SUSv4-2018
  - there were also older POSIX and SUS standards

## `man feature_test_macros`

- modern source code should use
  - `_POSIX_C_SOURCE` for POSIX stuff (which includes C99)
  - `_XOPEN_SOURCE` for SUS stuff (POSIX plus XSI/XCurses)
  - `_DEFAULT_SOURCE` for POSIX plus BSD/SysV stuff
    - it is implied when `-std=` is not given
  - `_GNU_SOURCE` for all above plus a little more deprecated stuff plus GNU
    extensions

## Large File Support

- `ftell` variants, as an example
  - `long int ftell(FILE *);`
  - `off_t ftello(FILE *);`
  - `off64_t ftello64(FILE *);`
- `ftell` will fail when the file position cannot be represented by `long int`
  - normally `long int` is 64-bit for 64-bit programs and is not an issue
- `ftello` is defined to `ftello64` when `_FILE_OFFSET_BITS=64` is defined
  - define the macro and do not use `ftello64` directly

## POSIX Headers

- these C standard headers are extended
  - `assert.h`
  - `complex.h`
  - `ctype.h`
  - `errno.h`
  - `fenv.h`
  - `float.h`
  - `inttypes.h`
  - `iso646.h`
  - `limits.h`
  - `locale.h`
  - `math.h`
  - `setjmp.h`
  - `signal.h`
  - `stdarg.h`
  - `stdbool.h`
  - `stddef.h`
  - `stdint.h`
  - `stdio.h`
  - `stdlib.h`
  - `string.h`
  - `tgmath.h`
  - `time.h`
  - `wchar.h`
  - `wctype.h`
- these are mandatory headers
  - `aio.h`, asynchronous input and output
  - `arpa/inet.h`, definitions for internet operations
  - `cpio.h`, cpio archive values
  - `dirent.h`, format of directory entries
  - `dlfcn.h`, dynamic linking
  - `fcntl.h`, file control options
  - `fnmatch.h`, filename-matching types
  - `glob.h`, pathname pattern-matching types
  - `grp.h`, group structure
  - `iconv.h`, codeset conversion facility
  - `langinfo.h`, language information constants
  - `monetary.h`, monetary types
  - `net/if.h`, sockets local interfaces
  - `netdb.h`, definitions for network database operations
  - `netinet/in.h`, Internet address family
  - `netinet/tcp.h`, definitions for the Internet Transmission Control Protocol (TCP)
  - `nl_types.h`, data types
  - `poll.h`, definitions for the poll() function
  - `pthread.h`, threads
  - `pwd.h`, password structure
  - `regex.h`, regular expression matching types
  - `sched.h`, execution scheduling
  - `semaphore.h`, semaphores
  - `strings.h`, string operations
  - `sys/mman.h`, memory management declarations
  - `sys/select.h`, select types
  - `sys/socket.h`, main sockets header
  - `sys/stat.h`, data returned by the stat() function
  - `sys/statvfs.h`, VFS File System information structure
  - `sys/times.h`, file access and modification times structure
  - `sys/types.h`, data types
  - `sys/utsname.h`, system name structure
  - `sys/un.h`, definitions for UNIX domain sockets
  - `sys/wait.h`, declarations for waiting
  - `tar.h`, extended tar definitions
  - `termios.h`, define values for termios
  - `unistd.h`, standard symbolic constants and types
  - `wordexp.h`, word-expansion types
- these are xsi headers
  - `fmtmsg.h`, message display structures
  - `ftw.h`, file tree traversal
  - `libgen.h`, definitions for pattern matching functions
  - `ndbm.h`, definitions for ndbm database operations
  - `search.h`, search tables
  - `sys/ipc.h`, XSI interprocess communication access structure
  - `sys/msg.h`, XSI message queue structures
  - `sys/resource.h`, definitions for XSI resource operations
  - `sys/sem.h`, XSI semaphore facility
  - `sys/shm.h`, XSI shared memory facility
  - `sys/time.h`, time types
  - `sys/uio.h`, definitions for vector I/O operations
  - `syslog.h`, definitions for system error logging
  - `utmpx.h`, user accounting database definitions
- these are realtime headers
  - `mqueue.h`, message queues
  - `spawn.h`, spawn
- these are obsoleted
  - `stropts.h`, STREAMS interface
  - `trace.h`, tracing
  - `ulimit.h`, ulimit commands
  - `utime.h`, access and modification times structure

## Random

- PLT
  - when calling a global function `foo`, we actually call `foo@plt` which
    resolves the address of `foo` dynamically
  - if we define `foo`, and we want to call `foo` directly (e.g., `LD_PRELOAD`
    won't work anymore), glibc provides `libc_hidden_def` and
    `libc_hidden_proto`
    - both macros are no-ops outside of `libc.so`
    - they expand to `__attribute__ ((visibility ("hidden")))`
- `weak_alias(__foo, foo)` makes `foo` a weak alias of `__foo`
- a new syscall wrapper appears to be generated from
  `sysdeps/unix/sysv/linux/syscalls.list`
- `setvbuf` is defined by `weak_alias (_IO_setvbuf, setvbuf)`

