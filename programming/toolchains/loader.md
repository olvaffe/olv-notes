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
