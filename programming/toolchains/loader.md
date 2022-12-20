Dynamic Loader
==============

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
