Kernel KBuild
=============

## Top-level Makefile

- A top-level `make` will have these processes

    make
    make -f scripts/Makefile.build obj=arch/x86
    make -f scripts/Makefile.build obj=arch/x86/kernel
    make -f scripts/Makefile.build obj=arch/x86/kernel/apic
  - All subprocesses will have from `MAKEFLAGS` top-level `make` arguments
  - They will have all `export`ed variables
  - top make also adds `-rR --no-print-directory --include-dir=$(srctree)`
- It includes
  - `scripts/Kbuild.include`, a list of variables
  - `arch/x86/Makefile`, targets and arch sources
  - `include/config/auto.conf`, kernel configs
- All top directories are collected in `vmlinux-dirs`.
  - `$(Q)$(MAKE) $(build)=$@` is called on each of them to build all objects and
    `built-in.o`s.
  - `build` is expanded to `-f scripts/Makefile.build obj`
- Have a look at how `vmlinux.lds` is built as an example
  - arch makefile adds `arch/x86/` to `core-y`.
  - It is part of `vmlinux-dirs` and make is called with
    `-f scripts/Makefile.build obj=arch/x86/`
  - `arch/x86/Kbuild` is included
    - `Kbuild` is preferred over `Makefile` when exists.
  - `scripts/Makefile.lib` is included to transform `kernel/` in `obj-y` into
    - `kernel/` in `subdir-y`
    - `kernel/built-in.o` in `obj-y`
  - make is called again with `obj=arch/x86/kernel/`
  - `cmd_cpp_lds_S` is used to transform `%.lds.S` into `%.lds`.
- `make arch/x86`
  - This target is given as part of `vmlinux-dirs`
  - It runs `$(Q)$(MAKE) $(build)=arch/x86/`
  - See `d1f0ae5e2e45e74cff4c3bdefb0fc77608cdfeec`.

## Kconfig

- Top-level Makefile has wildcard target `%config`
- `include/config/auto.conf` and `include/linux/autoconf.h` is generated by
  `make silentoldconfig`
  - It executes `scripts/kconfig/conf`.

## Preparation Targets

- `prepare0`
  - it builds top-level Kbuild, which
  - generates `include/asm/asm-offsets.h` from `arch/x86/kernel/asm-offsets.c`.
  - checks missing syscalls
- `prepare1` depends
  - `include/config/auto.conf`, generated by `silentoldconfig`
  - `include/linux/version.h`, generatedy by makefile itself
  - `include/linux/utsrelease.h`, from `include/config/kernel.release`
  - `include/asm`, symlink to real arch asm directory

## Headers

- Top make runs `scripts/Makefile.headersinst` on `include/` and
  `arch/x86/include/`.

## `make help`

- cleaning targets
  - `clean` removes generated files but keeps `.config`
  - `mrproper` removes all generated files
- configuration targets
  - `alldefconfig` creates a new `.config` using defaults
    - `defconfig` is similar but is based on a arch-specific defconfig
  - `olddefconfig` updates `.config` by using defaults for new options
    - `oldconfig` is similar but prompts for new options
    - `config` is similar but prompts for all options
  - `menuconfig` updates `.config` with a ui
  - `savedefconfig` saves `.config` to `defconfig`, with options having
    default values omitted
- other generic targets
  - `all` or no target builds default targets
    - `vmlinux`, `modules`
    - if x86, `bzImage`
    - if arm64, `dtbs`, `Image.gz`
  - `vmlinux` builds `vmlinux`, the kernel in elf format
  - `modules` builds `*.ko`, all kernel modules in elf format
  - `modules_install` installs `*.ko` to
    `$INSTALL_MOD_PATH/lib/modules/$KERNELRELEASE`
    - `$INSTALL_MOD_PATH` defaults to `/`
    - it also invokes `depmod -b $INSTALL_MOD_PATH $KERNELRELEASE` to generate
      `modules.*`
  - `some/dir/` compiles `*.o` in a subdirectory (but not `*.ko`)
  - `modules_prepare` sets up for external modules
  - `tags` generates `tags`
  - `kernelversion` prints `$KERNELVERSION`, such as `6.11.2`
  - `kernelrelease` prints `$KERNELRELEASE`, which is
    `$KERNELVERSION$CONFIG_LOCALVERSION$CONFIG_LOCALVERSION_AUTO`
    - it is generated by `scripts/setlocalversion`
  - `image_name` prints `$KBUILD_IMAGE`
    - `arch/x86/boot/bzImage` on x86
    - `arch/arm64/boot/Image.gz` on arm64
  - `headers_install` installs uapi to `$INSTALL_HDR_PATH/include`
    - `$INSTALL_HDR_PATH` defaults to `/usr`
- devicetree targets
  - `dtbs` builds `*.dtb`
  - `dtbs_install` installs `*.dtb` to `$INSTALL_DTBS_PATH`
    - `$INSTALL_DTBS_PATH` defaults to `/boot/dtbs/$KERNELRELEASE`
- userspace tools targets
  - `acpi`, `bpf`, `perf`, etc.
- kernel packaging
  - `dir-pkg` generates `tar-install/`
  - `targz-pkg` generates `linux-$KERNELRELEASE-$ARCH.tar.gz`
  - `tarzst-pkg` generates `linux-$KERNELRELEASE-$ARCH.tar.zst`
- x86-specific targets
  - `bzImage` builds `bzImage`
- arm64-specific targets
  - `Image` builds `Image`
  - `Image.gz` builds `Image.gz`, gzipped `Image`
  - `image.fit` builds `image.fit`
    - it invokes `scripts/make_fit.py` to pack `Image` and `*.dtb` into a
      u-boot fit image
- variables
  - `V=1` be verbose
  - `O=dir` uses `dir` as the output director
  - `CHECK_DTBS=1` checks `*.dtb` against schemas
