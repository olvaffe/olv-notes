Meson
=====

## Usage

- `meson out; meson compile -C out`
- meson commands for the source dir
  - `setup` configures the build directory
    - this is the default command when none is given
    - `--wipe` wipes and reconfigures the build directory
    - `--reconfigure` reconfigures the build directory
    - `--native-file` specifies the native machine file
    - `--cross-file` specifies the cross machine file
  - `subprojects` manages bundled third-party projects
    - <https://mesonbuild.com/Subprojects.html>
    - `download`/`update`/`purge` sub-commands
    - `subprojects` directory contains wrap files and downloaded third-party
      projects
  - `wrap` manages wrap files for bundled third-party projects
    - <https://mesonbuild.com/Using-wraptool.html>
    - `list`/`search`/`info`/`install` sub-commands
  - `init` creates template `meson.build`
- meson commands for the build dir
  - `configure` modifies options
  - `compile` invokes the builder to build
    - `ninja`, `msbuild`, or `xcodebuild` depending on the OS
  - `install` invokes the builder to install
  - `dist` packages a release tarball
    - it is for the build dir because it runs tests before packaging
  - `test` runs tests
  - `introspect` displays info about a configured build directory

## Environment Variables

- `CC`, `CC_LD`, `CXX`, `CXX_LD`
  - not idiomatic
  - but more convenient than `--native-file` and a native machine flie
  - meson automatically uses ccache when it is available; to diable it, one
    must explicit override the compiler
- `CFLAGS`, `CXXFLAGS`, `LDFLAGS`
  - do not use if possible
  - use `c_args`, `cpp_args`, `c_link_args`, `cpp_link_args` instead

## Options

- `meson setup -D` or `meson configure -D` can set options
- core options
  - `buildtype` selects the buildtype
    - `plain` to not mess around with compile flags
  - `debug` and `optimization` are more flexible version of `buildtype`
  - `warning_level` and `werror` fine tune compiler warnings
  - `default_library` selects between static/shared
  - `strip` strips on install
- backend options
  - backend-specific options (e.g., ninja)
- base options
  - `b_ndebug` defines `NDEBUG` to disble `assert`
  - `b_asneeded` links with `-Wl,--as-needed`
  - `b_lundef` links with `-Wl,--no-undefined`
  - `b_lto` enables LTO
  - `b_pgo` enables PGO
  - `b_pie` enables `-pie`
  - `b_sanitize` enables sanitizers
- compiler options
  - `c_std`, `c_args`, `c_link_args`
  - `cpp_std`, `cpp_args`, `cpp_link_args`
  - `cpp_rtti`, `cpp_eh`
- directories
  - `prefix`, `bindir`, `datadir`, ...
- testing options
  - controls how tests are run
- project options
  - project-specific options

## Ninja

- `ninja` to build
- `ninja test` to run tests
- `ninja install` to install
  - `DESTDIR` is supported

## Machine Files

- machine files are used with `--native-file` or `--cross-file` to describe
  native or cross machines
  - there can be multiple machine files and they are concatenated together
- data types
  - integers (e.g., 42)
  - booleans (`true` or `false`)
  - strings (e.g., `'abc'`)
  - arrays of strings or booleans (e.g., `['abx', 'xyz']`)
- `[constants]` section
  - values defined in this section can be used in other sections
  - string concatenation with `+`
  - path concatenation with `/`
- `[binaries]` section
  - programs used by meson internally or returned by `find_program`
  - `c`, `c_ld`, `cpp`, `cpp_ld`, `ar`, `strip`, etc.
  - `pkgconfig`, `llvm-config`, etc.
- `[paths]` section
  - deprecated by `[built-in options]`
- `[properties]` section
  - `cmake_toolchain_file`
  - `java_home`
  - `<lang>_args` and `<lang>_link_args` are deprecated in favor of
    `[built-in options]`
- `[built-in options]` section
  - set built-in options such as `prefix`, `c_flags`, or `pkg_config_path`
  - use `[<subject>: built-in options]` for subproject options
- `[project options]` section
  - set project options
  - use `[<subject>: project options]` for subproject options
- `[cmake]` section
- cross machine specifics
  - `[binaries]` section
    - `exec_wrapper`
  - `[properties]` section
    - `sizeof_int`
    - `sys_root`, used to set `PKG_CONFIG_SYSROOT_DIR`
    - `pkg_config_libdir`, used to set `PKG_CONFIG_LIBDIR`
  - `[host_machine]` section
    - `system`
    - `cpu_family`
    - `cpu`
    - `endian`

## Meson Cross-Compilation

- `meson --cross-file <cross_file.txt>`
- target 64-bit arm

    [constants]
    toolchain_prefix = '/usr/bin/aarch64-linux-gnu-'
    sysroot = '...'
    common_flags = ['--sysroot', sysroot]

    [binaries]
    ar = toolchain_prefix + 'ar'
    c = toolchain_prefix + 'gcc'
    cpp = toolchain_prefix + 'g++'
    strip = toolchain_prefix + 'strip'
    pkgconfig = '/usr/bin/pkg-config'

    [built-in options]
    c_args = common_flags + []
    c_link_args = common_flags + []
    cpp_args = common_flags + []
    cpp_link_args = common_flags + []

    [properties]
    sys_root = sysroot
    pkg_config_libdir = sysroot + '/usr/lib/pkgconfig:' + sysroot + '/usr/share/pkgconfig'

    [host_machine]
    system = 'linux'
    cpu_family = 'aarch64'
    cpu = 'aarch64'
    endian = 'little'
- target 32-bit x86

    [constants]
    toolchain_prefix = '/usr/bin/'
    common_flags = ['-m32']

    [binaries]
    ar = toolchain_prefix + 'ar'
    c = toolchain_prefix + 'gcc'
    cpp = toolchain_prefix + 'g++'
    strip = toolchain_prefix + 'strip'
    pkgconfig = '/usr/bin/pkg-config'
    
    [built-in options]
    c_args = common_flags + []
    c_link_args = common_flags + []
    cpp_args = common_flags + []
    cpp_link_args = common_flags + []
    
    [properties]
    pkg_config_libdir = '/usr/lib32/pkgconfig:/usr/share/pkgconfig'

    [host_machine]
    system = 'linux'
    cpu_family = 'x86'
    cpu = 'i686'
    endian = 'little'
- target 32-bit arm
    [constants]
    toolchain_prefix = '/usr/bin/arm-linux-gnueabihf-'
    sysroot = '...'
    common_flags = ['--sysroot', sysroot]

    [binaries]
    ar = toolchain_prefix + 'ar'
    c = toolchain_prefix + 'gcc'
    cpp = toolchain_prefix + 'g++'
    strip = toolchain_prefix + 'strip'
    pkgconfig = '/usr/bin/pkg-config'

    [built-in options]
    c_args = common_flags + []
    c_link_args = common_flags + []
    cpp_args = common_flags + []
    cpp_link_args = common_flags + []

    [properties]
    sys_root = sysroot
    pkg_config_libdir = sysroot + '/usr/lib/pkgconfig:' + sysroot + '/usr/share/pkgconfig'

    [host_machine]
    system = 'linux'
    cpu_family = 'arm'
    cpu = 'armv7a'
    endian = 'little'
- x86-64
  - both `cpu_family` and `cpu` are `x86_64`
- there seems to be cases where `SYSROOT=<sysroot>` env var works better than
  `--sysroot` flag
