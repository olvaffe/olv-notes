BusyBox
=======

## busybox build system

- `scripts/gen_build_files.sh` parses the applets and generate various files
  - `//config` is combined with `Config.src` to generate `Config.in`
  - `//applet` is combined with `include/applets.src.h` to generate
    `include/applets.h`
  - `//kbuild` is combined with `Kbuild.src` to generate `Kbuild`
  - `//usage` is combined with `include/usage.src.h` to generate
    `include/usage.h`
    - there are several levels of usage docs
      - `trivial` is always included
      - `full` is included when `CONFIG_FEATURE_VERBOSE_USAGE`
      - `notes`, `example`, and `sample` are never included?

## toybox

- started from scratch by then busybox maintainer after licensing argument
