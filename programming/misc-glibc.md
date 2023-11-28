glibc
=====

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
