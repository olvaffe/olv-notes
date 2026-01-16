pkgconf
=======

## Overview

- <http://pkgconf.org/>
  - `pkgconf` is compatible with `pkg-config`
  - <https://www.freedesktop.org/wiki/Software/pkg-config/> is unmaintained

## Environmental Variables

- `PKG_CONFIG_LIBDIR` is a colon-separated list of primary search paths
  - it defaults to `/usr/lib/pkgconfig:/usr/share/pkgconfig`
  - use this for cross-compilation, to avoid finding the host pc files
- `PKG_CONFIG_PATH` is a colon-separated list of secondary search paths
  - it has no default
  - use this to add extra paths
- `PKG_CONFIG_SYSROOT_DIR` is the sysroot
  - the man page says that it affects `PKG_CONFIG_PATH`, but that does not
    seem true
  - from experiments, `PKG_CONFIG_SYSROOT_DIR` should be set to the absolute
    path of the sysroot and `PKG_CONFIG_LIBDIR` should be set to the absolute
    paths of search paths under the sysroot
  - the sysroot is also prepended to the `prefix` variable
