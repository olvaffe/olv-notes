GN
==

## Usage

- basics
  - `gn gen out/my_build`
  - `gn args out/my_build`
  - `gn args --list out/my_build`

## Cross-compile

- `gn args out/my_build`
  - `target_os = "chromeos"`
    - or `"linux"` or `"android"`
    - `gn help target_os`
  - `target_cpu = "arm64"`
    - or `"x86"` or `"x64"` or `"arm"`
    - `gn help target_cpu`
  - `target_cc = "/usr/bin/aarch64-linux-gnu-gcc"`
  - `target_cxx = "/usr/bin/aarch64-linux-gnu-g++"`
  - `extra_cflags = [ "--sysroot=<path>" ]`
  - `extra_ldflags = [ "--sysroot=<path>" ]`
