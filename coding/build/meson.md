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

## Define Options

- options are defined in `meson.options`
  - or `meson_options.txt`
- synax: `option('foo', type : 'bar', value : 'val', description : 'desc')`
  - `foo` is the name
  - `type` can be
    - `string`: a string
    - `boolean`: `true` or `false`
      - `true` is the default if no `value`
    - `combo`: one of the string specified in `choices` array
      - the first choice is the default if no `value`
    - `integer`: an integer
      - optional `min` and `max`  to specify the min/max valid values
    - `array`: an array of strings
      - optional `choices` array for valid strings
      - `choices` array is also the default if no `value`
    - `feature`: one of `enabled`, `disabled`, or `auto`
  - `value` is the default value
  - `description` is the description
  - `deprecated` marks an option deprecated
    - if `true`, the option is deprecated
    - if an array, choices in the array are deprecated
    - if a dict, choices in the dict keys are deprecated and are replaced by
      the corresponding dict values
    - if another option, the option is deprecated by the other option

## Dependencies

- to find an external dependency,
  - `dependency('zlib', version : '>=1.2.8')`
- to declare a dependency,
  - `declare_dependency(link_with : my_lib, include_directories : my_inc)`
- to be able to fall back to a subproject,
  - `dependency('foo', fallback : ['foo', 'foo_dep'])`
  - if `foo` is not available as an externaly dependency but available as a
    subproject, build `foo` and use `foo_dep` dependency from the subproject
    - `foo` must be a meson subproject with
      `foo_dep = declare_dependency(...)`
- detection methods
  - the default is `auto`
    - meson has special logics for certain external dependencies such as `dl`,
      `llvm`, `python3`, `qt5`, `sdl2`, `threads`, `zlib`, etc.
    - otherwise, meson tries `pkg-config` and then `cmake`
  - `pkg-config` uses `pkg-config`
  - `cmake` uses cmake's `find_package`
  - `system` uses `compiler.find_library()`
  - `builtin` checks if the external depenency is a compiler builtin
  - `config-tool` uses dependency-specific tools such as `llvm-config`
  - `sysconfig` uses python3's `sysconfig`
  - `qmake` uses qt's `qmake`

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
      - <https://mesonbuild.com/Reference-tables.html#operating-system-names>
      - `android`, `darwin`, `linux`, `windows`, etc.
      - `host_machine.system()` returns the string
    - `cpu_family`
      - <https://mesonbuild.com/Reference-tables.html#cpu-families>
      - `aarch64`, `arm`, `x86`, `x86_64`, etc.
      - `host_machine.cpu_family()` returns the string
    - `cpu`
      - `detect_cpu` in `mesonbuild/environment.py`
      - `aarch64`, `arm`, `i686`, `x86_64`, etc.
      - `host_machine.cpu()` returns the string
      - for arm 32-bit, `armv7a` is more common
    - `endian`
      - `host_machine.endian()` returns the string

## Meson Cross-Compilation

- `meson --cross-file <cross_file.txt>`
- target 64-bit arm

    [constants]
    toolchain_prefix = '/usr/bin/aarch64-linux-gnu-'
    sysroot = '...'
    common_flags = ['--sysroot=' + sysroot]

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
    common_flags = ['--sysroot=' + sysroot]

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
