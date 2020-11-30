# Build

## Meson

 - mkdir buildir; cd builddir; meson ..
   - "--buildtype=debug" by default
   - "--prefix=/usr/local" by default
   - "--buildtype=plain" to not mess with the compiler flags
 - Options
   - prefix is an option
   - c_args, c_link_args, c_std are options
   - cpp_args, cpp_link_args, cpp_std are options
   - projects can add more options
   - see all options by running "meson configure"
 - environment variables
   - CC, CFLAGS, CPPFLAGS, LDFLAGS, CXX, CXXFLAGS are supported

## Ninja

 - "ninja" to build
 - "ninja test" to run tests
 - "ninja install" to install
   - "DESTDIR" is supported

## Meson Cross-Compilation

 - `meson --cross-file <cross_file.txt>`
 - target 32-bit x86
    [binaries]
    c = '/usr/bin/gcc'
    cpp = '/usr/bin/g++'
    ar = '/usr/bin/gcc-ar'
    strip = '/usr/bin/strip'
    pkgconfig = '/usr/bin/pkg-config'
    
    [properties]
    c_args = ['-m32']
    c_link_args = ['-m32']
    cpp_args = ['-m32']
    cpp_link_args = ['-m32']
    
    [host_machine]
    system = 'linux'
    cpu_family = 'x86'
    cpu = 'i686'
    endian = 'little'
 - since 32-bit x86 shares pkg-config with 64-bit x86, `PKG_CONFIG_PATH`
   should be specified
 - target 32-bit arm
    [binaries]
    ar = '/usr/bin/$HOST-ar'
    c = '/usr/bin/$HOST-clang'
    cpp = '/usr/bin/$HOST-clang++'
    strip = '/usr/bin/$HOST-strip'
    pkgconfig = '$SYSROOT/build/bin/pkg-config'
    
    [properties]
    c_args = ['--sysroot=$SYSROOT', ...]
    c_link_args = ['--sysroot=$SYSROOT', '-Wl,-O2', '-Wl,--as-needed']
    cpp_args = ['--sysroot=$SYSROOT', ...]
    cpp_link_args = ['--sysroot=$SYSROOT', '-Wl,-O2', '-Wl,--as-needed']
    
    [host_machine]
    system = 'linux'
    cpu_family = 'arm'
    cpu = 'armv7a'
    endian = 'little'
