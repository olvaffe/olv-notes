OpenWrt
=======

## Build

- build
  - `git clone https://git.openwrt.org/openwrt/openwrt.git`
  - `./scripts/feeds update -a`
    - `parse_config` parses `feeds.conf.default`
    - `update_feed` updates the listed repos
      - `update_location` writes the repo url to `feeds/$repo.tmp/location`
      - `update_feed_via` clones the repo to `feeds/$repo`
    - `update_index` creates the index files
      - `make prepare-mk TMP_DIR=feeds/$repo.tmp`
      - `make -f include/scan.mk SCAN_TARGET=packageinfo SCAN_DIR=feeds/$repo TMP_DIR=feeds/$repo.tmp`
        - this creates `feeds/$repo.tmp/.packageinfo`
      - `make -f include/scan.mk SCAN_TARGET=targetinfo SCAN_DIR=feeds/$repo TMP_DIR=feeds/$repo.tmp`
        - this creates `feeds/$repo.tmp/.targetinfo`
      - `ln -sf feeds/$repo.tmp/.packageinfo feeds/$repo.index`
      - `ln -sf feeds/$repo.tmp/.targetinfo feeds/$repo.targetindex`
  - `./scripts/feeds install -a`
    - `get_installed` creates and parses the index files
      - `make prepare-tmpinfo` creates `tmp/.packageinfo` by scanning
        `package/` and `tmp/.targetinfo` by scanning `target/linux/`
      - `parse_package_metadata` parses `tmp/.packageinfo`
      - `get_targets` parses `tmp/.targetinfo`
    - `get_feed` creates symlinks
      - `package/feeds/foo/bar` points to `feeds/foo/baz/bar`
  - `./scripts/feeds install -a`
  - get the default config for the target/subtarget
    - `wget -O .config https://downloads.openwrt.org/snapshots/targets/mediatek/filogic/config.buildinfo`
      for mediatek/filogic
  - `make menuconfig` to update `.config`
  - `make download` to download dependencies to `dl/`
    - the biggest ones are `linux-firmware`, `linux`, `llvm-project`, `gcc`,
      `binutils`, `gdb`, `u-boot`, etc.
  - `make`
    - the default target, the first one listed in `Makefile`, is `world`
    - `world` depends on `prepare`
      - `prepare` depends on `$(tools/stamp-compile)`
        - `$(curdir)/builddirs-default` in `tools/Makefile` defines all enabled
          packages
        - it builds all packages with `make -C tools/foo compile`
          - packages are built in `build_dir/host/` and installed to
            `staging_dir/host/`
      - `prepare` depends on `$(toolchain/stamp-compile)`
        - `$(curdir)/builddirs` in `toolchain/Makefile` defines all enabled
          packages
        - it builds all packages with `make -C toolchain/foo compile`
          - packages are built in `build_dir/toolchain-foo/` and installed to
            `staging_dir/toolchain-foo/`
    - `world` depends on `$(target/stamp-compile)`
      - `$(curdir)/builddirs-default` in `target/Makefile` defines all enabled
        targets
        - there is only `linux`
      - it builds the kernel with `make -C target/linux compile`
        - kernel is built in `build_dir/target-foo/`
    - `world` depends on `$(package/stamp-compile)`
    - `world` depends on `$(target/stamp-install)`
    - `world` depends on `$(package/stamp-install)`
    - the final images and packages are under `bin/`
- cleanup
  - `make clean` cleans target and package
  - `make targetclean` additionally cleans toolchain
  - `make dirclean` additionally cleans tools
- working with a single package
  - `make foo/bar/compile`
  - `make foo/bar/clean`
  - `V=s` for verbose

## Config

- the top-level config is `Config.in`
  - `source "target/Config.in"`
    - this sources the generated `tmp/.config-target.in`
  - `source "config/Config-images.in"`
    - this sources `target/linux/*/image/Config.in` as well
  - `source "config/Config-build.in"`
  - `source "config/Config-devel.in"`
  - `source "toolchain/Config.in"`
    - this sources `toolchain/binutils/Config.in`,
      `toolchain/gcc/Config.in`, and `toolchain/musl/Config.in` as well
  - `source "target/imagebuilder/Config.in"`
  - `source "target/sdk/Config.in"`
  - `source "target/toolchain/Config.in"`
  - `source "tmp/.config-package.in"`
    - it includes `package/*/image-config.in` as well
      - this expands to `package/base-files/image-config.in`, which includes
        `tmp/.config-feeds.in`

## Hardware

- Minimum Requirements
  - 128MB ram
  - 16MB flash
- brands
  - TP-LINK
  - NETGEAR
  - D-Link
  - Linksys
  - ASUS
  - Ubiquiti
- Wi-Fi 6 (IEEE 802.11ax)
  - TP-LINK Archer AX????
    - soc: BCM6750 (ARMv7), BCM6755 (ARMv7), BCM4908 (ARMv8)
    - ram: 256-1024MB
    - flash: 16-512MB
  - NETGEAR Nighthawk AX8
    - soc: BCM4908
    - ram: 1024MB
    - flash: 512MB
  - NETGEAR Orbi
    - soc: IPQ8074A (quad core A53)
    - ram: 1024MB
    - flash: 512MB
  - D-LINK
    - soc: MT7981
    - ram: 512MB
    - flash: 128MB
  - Linksys
    - soc: IPQ6018
    - ram: 512MB
    - flash: 512MB
  - ASUS \*-AX\*
    - soc: BCM6755, BCM4908, BCM4912
    - ram: 1024-2048MB
    - flash: 256MB
  - Ubiquiti UniFi AP 6
    - soc: MT7621AT (mips), IPQ5018 (dual core A53)
    - ram: 256-512MB
    - flash: 32-128MB
  - Nest Wifi Pro
    - Qualcomm Immersive Home 316 Platform
      - IPQ5018 with dual-core A53 @ 1GHz
    - 1GB DDR3L
    - 4GB eMMC
- Wi-Fi 7 (IEEE 802.11be)
  - TP-LINK Archer BE800
    - Qualcomm Networking Pro 1220
      - soc: quad core A73 2.2GHz
    - ram: 2048MB
    - flash: 256MB
  - NETGEAR Nighthawk RS700S
    - soc: quad core 2.6GHz
    - ram: 2048MB
    - flash: 512MB
- <https://hackerboards.com/>
  - Raspberry Pi
    - 4: BCM2711
    - 5: BCM2712
  - FriendlyElec NanoPi
    - R2S: RK3328
    - R4S: RK3399
    - R6S: RK3588S
  - Sinovoip Banana Pi
    - BPI-R3: MT7986
    - BPI-R4: MT7988A
    - BPI-M7: RK3588
  - Xunlong Orange Pi
    - 3: RK3566
    - 5: RK3588
  - Raxda Rock
    - 5B: RK3588
