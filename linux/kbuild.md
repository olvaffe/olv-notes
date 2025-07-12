Kernel KBuild
=============

## Overview

- `scripts/Makefile.build` is the main kbuild makefile
  - it is used to build the kernel
  - top-level `Makefile` is not a part of kbuild, but provides various goals
    where some of them use kbuild to build tke kernel
- upon `make`,
  - `__sub-make:` invokes a submake with
    `make -C $(abs_output) -f $(abs_srctree)/Makefile`
  - `$(build-dir): prepare` invokes another submake with
    `make -f ../scripts/Makefile.build obj=.`
    - `$(build)=$@` expands to `-f ../scripts/Makefile.build obj=.`
    - this tells kbuild to build `.`, and because kbuild prioritizes `Kbuild`
      over `Makefile`, and it will use top-level `Kbuild` rather than
      `Makefile`
    - top-level `Kbuild` sets `obj-y` to various top dirs such as `init/`,
      `usr/`, `arch/$(SRCARCH)/`, etc.
    - `scripts/Makefile.lib` extracts subdirs from `obj-y` to `subdir-ym`
  - `$(subdir-ym):` invokes yet another submake with
    `make -f ../scripts/Makefile.build obj=<foo>`
    - this visits all subdirs in `obj-y` recursively
    - each subdir is built independent from each other with a submake
- variables
  - `$(CURDIR)` is the output dir
    - it is from make and is by definition the abs path of the current dir,
      which is always the output dir
  - `$(objtree)` is the output dir
    - it may be abs, or rel to the current dir (thus just `.`)
  - `$(obj)` is the output subdir
    - kbuild expand `$(build)=$@` to `-f $(srctree)/scripts/Makefile.build obj=$@`,
      where `$@` is a subdir listed in `obj-y` or `obj-m`
    - it is thus always rel to the output dir
  - `$(srctree)` is the top-level `Makefile` dir
    - it may be abs, or rel to the current dir
  - `$(srcroot)` is `$(srctree)`, or the top-level external module dir
  - `$(src)` is `$(srcroot)/$(obj)`

## `make`

- the default goals are
  - `__all:` is the default goal
  - `__all: all` adds `all`
  - `all: vmlinux` adds `vmlinux`
  - `all: bzImage` adds `bzImage`
  - more
- `make vmlinux`
  - `vmlinux: vmlinux.o $(KBUILD_LDS) modpost`
  - `vmlinux.o modules.builtin.modinfo modules.builtin: vmlinux_o`
  - `vmlinux_o: vmlinux.a $(KBUILD_VMLINUX_LIBS)`
  - `vmlinux.a: $(KBUILD_VMLINUX_OBJS) scripts/head-object-list.txt FORCE`
  - `KBUILD_VMLINUX_OBJS := ./built-in.a`
  - `$(obj)/built-in.a: $(real-obj-y) FORCE`
  - `real-obj-y := $(call real-search, $(obj-y), .o, -objs -y)`
- `kbuild-file = $(or $(wildcard $(src)/Kbuild),$(src)/Makefile)`
  - this is how `Kbuild` is preferred over `Makefile`
- submakes
  - `__all: __sub-make`
  - `__sub-make:` invokes make again from the output dir
  - `$(build-dir): prepare` invokes make again with a different makefile
    - `build := -f $(srctree)/scripts/Makefile.build obj`

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

## Prepare Targets

- `archprepare` depends on many targets
- `outputmakefile`
  - `ln -fsn $(srcroot) source` creates a symlink to `$(srcroot)`
  - `$(call cmd,makefile)` runs `cmd_makefile` to generate `Makefile`
- `archheaders`
  - `arch/x86/Makefile` generates syscall headers to
    `arch/$(SRCARCH)/include/generated`
- `archscripts`
- `scripts` builds `scripts/basic`, `scripts/dtc`, and `scripts/`
- `include/config/kernel.release`
  - `filechk_kernel.release` invokes `scripts/setlocalversion` to generate the
    release version string
- `asm-generic` generates asm headers to
  - `arch/$(SRCARCH)/include/generated/asm`
  - `arch/$(SRCARCH)/include/generated/uapi/asm`
- `$(version_h)` generates `include/generated/uapi/linux/version.h`
  - `filechk_version.h` generates `LINUX_VERSION_*` and `KERNEL_VERSION`
    macros
- `include/generated/utsrelease.h`
  - `filechk_utsrelease.h` generates `UTS_RELEASE` macro
- `include/generated/compile.h`
  - `filechk_compile.h` invokes `scripts/mkcompile_h` to generate
    `UTS_MACHINE` and `LINUX_COMPILE*` macros
- `include/generated/autoconf.h`
  - Top-level Makefile has wildcard target `%config`
  - `make synconfig` invokes `scripts/kconfig/conf` to generate
    - `include/config/auto.conf` is similar to `.config`, but is sourced by
      `make` to define `CONFIG_RFKILL=y`, `CONFIG_DRM_GPUVM=m`, etc.
    - `include/config/auto.conf.cmd` is makefile rules to regenerate
      `include/config/auto.conf` when any of `Kconfig` changes
    - `include/generated/autoconf.h` is similar to `.config`, but is included
      by C source files
      - `CONFIG_RFKILL=y` becomes `#define CONFIG_RFKILL 1`
      - `CONFIG_DRM_GPUVM=m` becomes `#define CONFIG_DRM_GPUVM_MODULE 1`

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
      (`arch/$(SRCARCH)/configs/$(KBUILD_DEFCONFIG`)
  - `olddefconfig` updates `.config` by using defaults for new options
    - `oldconfig` is similar but prompts for new options
    - `config` is similar but prompts for all options
  - `menuconfig` updates `.config` with a ui
  - `savedefconfig` saves `.config` to `defconfig`, with options having
    default values omitted
- build targets
  - `vmlinux` builds `vmlinux`, the kernel in elf format
  - `modules` builds `*.ko`, all kernel modules in elf format
  - `some/dir/` compiles `*.o` in a subdirectory (but not `*.ko`)
- devicetree targets
  - `dtbs` builds `*.dtb`
  - `dt_binding_check` validates schemas under `Documentation/devicetree/bindings`
  - `dtbs_check W=1` validates dtbs using schemas
- install targets
  - `install` invokes distro-specific `installkernel` to install kernel image
  - `modules_install` installs `*.ko` to
    `$INSTALL_MOD_PATH/lib/modules/$KERNELRELEASE`
    - `$INSTALL_MOD_PATH` defaults to `/`
    - it also invokes `depmod -b $INSTALL_MOD_PATH $KERNELRELEASE` to generate
      `modules.*`
  - `headers_install` installs uapi to `$INSTALL_HDR_PATH/include`
    - `$INSTALL_HDR_PATH` defaults to `/usr`
  - `dtbs_install` installs `*.dtb` to `$INSTALL_DTBS_PATH`
    - `$INSTALL_DTBS_PATH` defaults to `/boot/dtbs/$KERNELRELEASE`
- packaging targets
  - `dir-pkg` generates `tar-install/`
  - `targz-pkg` generates `linux-$KERNELRELEASE-$ARCH.tar.gz`
  - `tarzst-pkg` generates `linux-$KERNELRELEASE-$ARCH.tar.zst`
- other targets
  - `all` or no target builds default targets
    - `vmlinux`, `modules`
    - if x86, `bzImage`
    - if arm64, `dtbs`, `Image.gz`
  - `modules_prepare` sets up for external modules
  - `tags` generates `tags`
  - `kernelversion` prints `$KERNELVERSION`, such as `6.11.2`
  - `kernelrelease` prints `$KERNELRELEASE`, which is
    `$KERNELVERSION$CONFIG_LOCALVERSION$CONFIG_LOCALVERSION_AUTO`
    - it is generated by `scripts/setlocalversion`
  - `image_name` prints `$KBUILD_IMAGE`
    - `arch/x86/boot/bzImage` on x86
    - `arch/arm64/boot/Image.gz` on arm64
- userspace tools targets
  - `acpi`, `bpf`, `perf`, etc.
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

## Build Flow

- steps
  - `make alldefconfig` or `make olddefconfig`
    - `alldefconfig` generates `.config` using defaults
    - `olddefconfig` updates `.config` to sets new options to defaults
  - `make menuconfig` edits `.config`
  - `make` builds kernel image, modules, and dtbs
- cross compile
  - get cross compiler
    - `crossbuild-essential-arm64` on Debian-like
    - `aarch64-linux-gnu-gcc` on Arch-like
  - set variables
    - `ARCH=arm64`
    - `CROSS_COMPILE=aarch64-linux-gnu-`
- post install (Arch)
  - initramfs
    - `mkinitcpio -k $version -g /boot/initramfs.img`
  - systemd-boot
    - `/boot/loader/entries/custom.conf`
      - `title Custom Linux`
      - `linux /vmlinuz`
      - `initrd /initramfs.img`
      - `options root=/dev/sda2 rw`
  - out-of-tree drivers
    - `pacman -S broadcom-wl-dkms`
    - `dkms autoinstall`
- post install (Raspberry Pi)
  - copy `arch/arm64/boot/Image` and
    `arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb`
  - `config.txt`
    - `arm_64bit=1`
    - `kernel=Image`

## External (Out-of-Tree) Modules

- <https://docs.kernel.org/kbuild/modules.html>
- steps
  - build the kernel first
    - `make O=TEST`
  - build the external module
    - `make -C TEST M=<path-to-ext-module>`
    - the default target is `modules`
- `Makefile` and `Kbuild`
  - the external module typically contains `Makefile` for convenience
    - it allows users to invoke `make` to build the ext mod
    - `KDIR ?= /lib/modules/$(uname -r)/build`
    - `default:`
    - ` $(MAKE) -C $(KDIR) M=$$PWD`
  - there is a separate `Kbuild` to build the external module
    - `obj-m := foo.o`
    - `foo-y := foo-core.o foo-plat.o`
  - since 6.13, `Makefile` can instead
    - `KDIR ?= /lib/modules/$(uname -r)/build`
    - `export KBUILD_EXTMOD := $(realpath $(dir $(lastword $(MAKEFILE_LIST))))`
    - `include $(KDIR)/Makefile`
  - it is possible to merge `Kbuild` into `Makefile`
    - `$(KERNELRELEASE)` is non-empty when kbuild sources `Makefile`
    - `ifneq ($(KERNELRELEASE),) ... else ... endif`
- symbol check
  - `modules: modpost`
  - `modpost:` invokes `$(MAKE) -f $(srctree)/scripts/Makefile.modpost`
  - `$(output-symdump):` generates `Module.symvers`
    - this runs `$(objtree)/scripts/mod/modpost` with
      - `-M` checks `EXPORT_SYMBOL`
      - `-o Module.symvers` outputs `Module.symvers`
        - each line is an exported symbol of a module
      - `-T modules.order`
        - `modules.order` is a list of all `.o`
        - this parses `.o` and checks symbols
    - if external module, also
      - `-i $(objtree)/Module.symvers`
        - this gets exported symbols from the kernel build
        - `KBUILD_EXTRA_SYMBOLS` specifies additional `Module.symvers`
      - `-e` specifies an external module
