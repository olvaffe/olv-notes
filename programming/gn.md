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
    - `gn help target_os`
  - `target_cpu = "arm64"`
    - `gn help target_cpu`
  - `target_cc = "..."`
  - `target_cxx = "..."`
- sysroot?
  - `extra_cflags = [ "--sysroot=<path>" ]`
  - `extra_ldflags = [ "--sysroot=<path>" ]`
