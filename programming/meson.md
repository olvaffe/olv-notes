Meson
=====

## Usage

- `mkdir buildir; cd builddir; meson ..`
  - `--buildtype=debug` by default
  - `--prefix=/usr/local` by default
  - `--buildtype=plain` to not mess with the compiler flags
- Options
  - `prefix` is an option
  - `c_args`, `c_link_args`, `c_std` are options
  - `cpp_args`, `cpp_link_args`, `cpp_std` are options
  - projects can add more options
  - see all options by running `meson configure`
- environment variables
  - `CC`, `CFLAGS`, `CPPFLAGS`, `LDFLAGS`, `CXX`, `CXXFLAGS` are supported

## Ninja

- `ninja` to build
- `ninja test` to run tests
- `ninja install` to install
  - `DESTDIR` is supported

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
- there seems to be cases where `SYSROOT=<sysroot>` env var works better than
  `--sysroot` flag
