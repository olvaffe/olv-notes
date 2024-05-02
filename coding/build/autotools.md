Autotools
=========

## Autoconf

- meaing of arguments passed to `AC_INIT`

        AC_INIT(PACKAGE-NAME, VERSION, BUG-REPORT, [TARNAME])
        
        #define PACKAGE_NAME "PACKAGE_NAME"
        #define PACKAGE_VERSION "VERSION"
        #define PACKAGE_BUGREPORT "BUG-REPORT"
        #define PACKAGE_TARNAME (default to PACKAGE_NAME with "GNU " prefix stripped)
        #define PACKAGE_STRING "PACKAGE_NAME VERSION"
        
        #define PACKAGE "PACKAGE_TARNAME"
        #define VERSION "PACKAGE_VERSION"

## Automake

- libs

        LDADD: target is linked to specified libs
        LIBADD: target depends on specified libs
        
        when target == PROGRAM, use LDADD
        when target == LIBRARY, use LIBADD
- user configurable variables and am variants
  - user's ones are given in the commandline
  - am's ones are given in Makefile.am

        LIBTOOLFLAGS    -> AM_LIBTOOLFLAGS
        CPPFLAGS        -> AM_CPPFLAGS
        CFLAGS          -> AM_CFLAGS
        LDFLAGS          -> AM_LDFLAGS
- `INCLUDES` is deprecated in favor of `AM_CPPFLAGS` and `per-target_CPPFLAGS`

## libtool

- add `AC_PROG_LIBTOOL` in `configure.ac`.  `./config.status` generates
  `libtool`
  - The first section of `libtool` is configs
  - Following is the contents of `ltmain.sh`, adjusted for different shells
- when `--mode==link`, options no listed in `libtool --mode=link --help` are
  ignored. (not true, because some other options are also passed)
- 

## cross-compiling

- `host` is the arch the program runs on
- `build` is the arch on which the program is built
- `target` is the arch the program generates for
  - e.g., the arch gcc generates machine code for
- `target` defaults to `host`, `host` defaults to `build`, `build` defaults to
  `config.guess`
- a computer system is named using a configuration name
  - sometimes called configuration triplets
  - `<cpu>-<manufacturer>-<os>`
  - `<cpu>-<manufacturer>-<kernel>-<os>`
  - manufacturer is optional
- valid configuration names are defined by gcc
  - i.e., gcc targets
- cpu
  - `i386`, `i586`, `i686`, `x86_64`, `arm`
- manufacturer
  - `unknown`
- kernel
  - `linux`
- os
  - `winnt4.0`, `elf`, `coff`, `gnu`, `gnueabi`, `eabi`, `android`

