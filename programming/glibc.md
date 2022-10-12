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
