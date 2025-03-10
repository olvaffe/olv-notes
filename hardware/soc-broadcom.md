Broadcom SoC
============

## Raspberry Pi History

- RPi 1 uses BCM2835
  - VideoCore IV, 250MHz
  - ARM1176 (ARMv6)
- RPi 2 uses BCM2836
  - VideoCore IV, 250MHz
  - Cortex-A7 (ARMv7)
- RPi 3 uses BCM2837
  - VideoCore IV, 300MHz
  - Cortex-A53 (ARMv8)
- RPi 4 uses BCM2711
  - Video Core VI, 500MHz
  - Cortex-A72 x4 @1.8GHz (ARMv8)
- RPi 5 uses BCM2712
  - Video Core VII, 800MHz
  - Cortex-A76 x4 @2.4GHz (ARMv8.2-A)

## Power Consumption

- RPi 4 peaks at ~8W and idles at ~2.4W
- RPi 5 peaks at ~12W and idles at ~2W
  - requires active cooling to avoid throttling
  - when using a 15W power adapter, it caps the downstream usb port to 600mA
    (3W)
  - when using a 25W power adapter, it caps the downstream usb port to 1.6A
    (8W)

## Raspberry Pi: First and Second Stage Bootloaders

- <https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#eeprom-boot-flow>
- VideoCore has two main processors with differnet instruction sets
  - a VPU, dual-core dual-issue 16-way SIMD for system managements, codecs,
    etc.
  - QPUs, for 3D works
  - Linux runs on ARM CPU
- When powered on, VPU executes stage1 bootloader from SoC ROM (bootrom)
  - CPU is reset and SDRAM is disabled
  - VPU initializes SD/eMMC controller
  - VPU looks for `recovery.bin` on the first FAT partition
    - if gpt, the partition type must be ESP or MS basic data
    - `recovery.bin` is a minimal stage2 bootloader
    - its job is to reflash the real stage2 bootloader to SPI EEPROM
  - VPU looks for the stage2 bootloader in SPI EEPROM
    - this is the normal boot flow
  - otherwise, it waits and loads `recovery.bin` from USB
- VPU executes stage2 bootloader from EEPROM
  - VPU initializes more of the system, including clocks and SDRAM
  - VPU checks the config on EEPROM which affects the boot flow
    - it can load the firmware locally from SD card, USB storage, or NVMe to
      SDRAM
      - if gpt, the partition type must be ESP or MS basic data
    - it can also load the firmware over network or USB (by entering usb
      gadget mode) to SDRAM
    - if it finds `pieeprom.upd` on local storage, it updates the stage2
      bootloader on EEPROM
- <https://github.com/raspberrypi/rpi-eeprom> provides tools to update
  the second stage bootloader and to edit bootloader configs
- troubleshooting
  - if the bootloader fails to transition to the firmware, it should show a
    diagnostics screen on HDMI output
    - this can be disabled by `DISABLE_HDMI=1`
    - the bootloader seems picky on the monitor though
  - `BOOT_UART=1` instructs the bootloader to output to uart
  - `BOOT_ORDER` selects the boot order
    - 1 for sd, 2 for pxe, 4 for usb, e to stop, f to restart from first entry
    - default is 0xf41; reading from right to left, it means finding from sd
      then usb with a indefinite loop

## Raspberry Pi: Firmware

- at the end of stage2 bootloader, VPU loads the VPU firmware
 - on legacy rpis, VPU loads `bootcode.bin` which chainloads `start.elf`
 - on rpi4, VPU loads `start4.elf`
 - on rpi5, no loading is needed because the firmware is embedded in stage2
- VPU executes the firmware (`start4.elf`) from SDRAM
  - the firmware is a full RTOS based on ThreadX
  - it parses `config.txt`, with these default values
    - `cmdline=cmdline.txt`, which reads kernel cmdline from `cmdline.txt`
    - `kernel=kernel8.img`, which loads kernel from `kernel8.img` (armv8)
    - `enable_uart=0`, which disables uart by default
    - `armstub=?`, which uses armstub embedded in `start4.elf` by default
      - <https://github.com/raspberrypi/tools/tree/master/armstubs>
      - armstub starts in EL3, performs initializations, switches to EL2, and
        jumps to the kernel
      - on rpi4, `armstub8.S` is used, corresponding to `kernel8.img`
  - it loads various AP (ARM CPU) images
  - ARM CPU is out of reset and starts running
- CPU executes some initializations, armstub, and kernel
- troubleshooting
  - the firmware shows a rainbow splash screen
    - this can be disabled by `disable_splash=1`
  - `enable_uart=1` enables `/dev/ttyS0` for the kernel
- <https://github.com/raspberrypi/firmware/> is the official
  proprietary/prebuilt firmwares
  - on rpi0 to rpi3, `bootcode.bin` parses `config.txt` and loads one of the
    gpu (VideoCore) fw
    - `start.elf` is regular
    - `start_x.elf` includes extra codecs
    - `start_db.elf` includes debug
    - `start_cd.elf` is cut-down
  - on rpi4, eeprom fw parses `config.txt` and loads one of the `start4*.elf`
    gpu (VideoCore) fw
  - `fixup*.dat` are the corresponding memory layout data files?
- <https://github.com/christinaa/rpi-open-firmware> is a semi-working open
  source firmware
  - it provides vpu stage2 bootloader and firmware
  - it initializes the system similar to stage2
  - it initializes ARM to run an embedded chainloader
  - the chainloader (re-)initializes eMMC controller and loads linux kernel
  - most hw blocks do not work
- <https://github.com/pftf/RPi4> is an UEFI firmware
  - it is built from official EDK2 and backed by ARM and VMWare
  - it specifies `armstub=RPI_EFI.fd` in `config.txt`, which instructs the vpu
    firmware to use `RPI_EFI.fd` instead of the built-in one as armstub
  - `RPI_EFI.fd` can chainload any EFI bootloader (grub2, systemd-boot, linux
    kernel itself)
- <https://source.denx.de/u-boot/u-boot.git> is u-boot
  - `make rpi_4_defconfig`
  - copy `u-boot.bin` to the boot dir
  - specify `kernel=u-boot.bin` in `config.txt`

## Minimal Boot Partition

- <https://github.com/raspberrypi/firmware>
  - copy these to esp
    - `boot/start4.elf`
    - `boot/fixup4.dat`
  - if downstream kernel,
    - copy these to esp too
      - `boot/kernel8.img`
      - `boot/bcm2711-rpi-4-b.dtb`
    - `cp -r modules/<version>-v8+ /usr/lib/modules`
    - downstream kernel is needed for eeprom bootloader update
- create `config.txt` on esp
  - `auto_initramfs=1`, to load `initramfs8` as initramfs
  - `enable_uart=1`, to enable uart
- create `cmdline.txt` on esp
  - `console=ttyS0,115200 console=tty0 root=PARTUUID=... rootwait`
    - note that `UUID=...` only works with initramfs
- when using distro kernel, we need postinst hooks to copy installed kernel as
  `kernel8.img`, copy `bcm2711-rpi-4-b.dtb`, and copy generated initramfs as
  `initramfs8`

## Distros

- <https://www.raspberrypi.com/software/> is the official distro
  - `2023-12-11-raspios-bookworm-arm64-lite.img.xz` is a compressed disk image
  - MBR partition table
    - partition 1: 512MB, VFAT, mounted to `/boot/firmware`
    - partition 2: 2GB, EXT4, mounted to `/`
  - `/boot/firmware` has all the vpu firmwares, kernels, initramfs, dtbs, etc.
    - `config.txt` has
      - `dtparam=audio=on`
      - `camera_auto_detect=1`
      - `display_auto_detect=1`
      - `auto_initramfs=1`
      - `dtoverlay=vc4-kms-v3d`
      - `max_framebuffers=2`
      - `disable_fw_kms_setup=1`
      - `arm_64bit=1`
      - `disable_overscan=1`
      - `arm_boost=1`
    - `cmdline.txt` has
      - `console=serial0,115200 console=tty1`
      - `root=PARTUUID=4e639091-02 rootfstype=ext4 fsck.repair=yes rootwait`
      - `quiet init=/usr/lib/raspberrypi-sys-mods/firstboot`
  - `/etc/kernel/postinst.d/z50-raspi-firmware`
    - it copies the latest kernel (`/boot/vmlinuz-*`), initramfs
      (`/boot/initrd.img-*)`, dtbs (`/usr/lib/linux-image-*/broadcom`), and
      overlays (`/usr/lib/linux-image-*/overlays`) to `/boot/firmware` when
      it is installed
- <https://wiki.debian.org/RaspberryPi4> is debian
  - `20231109_raspi_4_bookworm.img.xz` is a compressed disk image
  - it is similar to the official distro
  - it has these firmwares
    - `bluez-firmware`
    - `firmware-brcm80211`
    - `raspi-firmware`
    - `wireless-regdb`
- <https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4> is arch
  - `raspberrypi-bootloader` provides vpu bootloader and firmwares under
    `/boot`
    - <https://github.com/raspberrypi/firmware>
  - `uboot-raspberrypi` provides u-boot under `/boot`
    - `/boot/config.txt` has a single line, `enable_uart=1`
    - `/boot/kernel8.img` is actually `u-boot.bin`
    - `/boot/{mkscr,boot.scr,boot.txt}` is the boot script
    - `/boot/bcm271*.dtb`
  - `firmware-raspberrypi` provides firmwares under
    - `/usr/lib/firmware/updates/brcm`
    - `/usr/lib/firmware/updates/cypress`
  - `linux-aarch64` provides the generic kernel
  - `linux-rpi` provides the rpi-specific downstream kernel
    - <https://github.com/raspberrypi/linux>
    - it conflicts with `linux-aarch64` and `uboot-raspberrypi`
  - `rpi4-eeprom` provides the eeprom tools
    - <https://github.com/raspberrypi/rpi-eeprom>
    - `rpi-eeprom-update -a` to schedule an eeprom update
    - `rpi-eeprom-config` to view/edit the config
  - `raspberrypi-utils` provides various tools
    - <https://github.com/raspberrypi/utils>

## `/sys/firmware/devicetree`

- DT is built at kernel build time but the bootloader can patch it at runtime
- Kernel 32-bit `5.4.79-1-ARCH` with `config.txt`
  - `gpu_mem=64`
  - `dtparam=krnbt=on`
  - `dtoverlay=vc4-kms-v3d-pi4`
  - `initramfs initramfs-linux.img followkernel`
- `dtc -I fs /sys/firmware/devicetree/base` shows...
- compatible: `raspberrypi,4-model-b`
- `chosen`
  - bootargs
  - linux,initrd-start
  - linux,initrd-end
  - etc
- `reserved-memory`
  - `linux,cma`
    - compatible: `shared-dma-pool`
    - driver: `CONFIG_DMA_CMA`
- `cpus`
  - enable-method: `brcm,bcm2836-smp`
  - `cpu@0`, `cpu@1`, `cpu@2`, `cpu@3`
    - enable-method: `spin-table`
    - this means the firmware should make secondary CPUs spin, and branch to
      `cpu-release-addr` after thek kernel writes the address of
      `secondary_holding_pen` to it
- `sd_vcc_reg`, `fixedregulator_3v3`, and `fixedregulator_5v0`
  - compatible: `regulator-fixed`
  - driver: `CONFIG_REGULATOR_FIXED_VOLTAGE`
- `sd_io_1v8_reg`
  - compatible: `regulator-gpio`
  - driver: `CONFIG_REGULATOR_GPIO`
- `clocks`
  - `clk-osc` and `clk-usb`
    - compatible: `fixed-clock`
    - driver: `CONFIG_COMMON_CLK`
- `clk-108M`
  - compatible: `fixed-clock`
  - driver: `CONFIG_COMMON_CLK`
- `arm-pmu`
  - compatible: `arm,cortex-a72-pmu`
  - driver: `CONFIG_HW_PERF_EVENTS`
- `timer`
  - compatible: `arm,armv8-timer`
  - driver: `CONFIG_ARM_ARCH_TIMER`
- `leds`
  - compatible: `gpio-leds`
  - driver: `CONFIG_LEDS_GPIO`
- `phy`
  - compatible: `usb-nop-xceiv`
  - driver: `CONFIG_NOP_USB_XCEIV`
- `emmc2bus`
  - compatible: `simple-bus`
  - `emmc2`
    - compatible: `brcm,bcm2711-emmc2`
    - driver: `CONFIG_MMC_SDHCI_IPROC`
    - this is the microSD card
- `v3dbus`
  - compatible: `brcm,2711-v3d`
  - driver: `downstream CONFIG_DRM_V3D`
  - the downstream v3d driver binds to this device and is render-only without
    KMS
- `gpu`
  - compatible: `brcm,bcm2711-vc5`
  - driver: `CONFIG_DRM_VC4`
  - the vc4 driver binds to this device, and it has many component drivers
    that bind to various other devices (DPI, DSI, HDMI, HVS, PixelValve, TXP,
    V3D, VEC).
  - the vc4 driver's v3d component driver cannot not drive the v3d device on
    RPi4 (it can on RPi 1,2,3).  As such, it is KMS-only without rendering.
- `scb`
  - compatible: `simple-bus`
  - `ethernet`
    - compatible: `brcm,bcm2711-genet-v5`
    - driver: `CONFIG_BCMGENET`
  - `pcie`
    - compatible: `brcm,bcm2711-pcie`
    - driver: `CONFIG_PCIE_BRCMSTB`
  - `xhci`
    - compatible: `generic-xhci`
    - driver: `CONFIG_USB_XHCI_PLATFORM`
    - status: `disabled` (because it is on PCIe bus, not on platform bus)
  - `dma`
    - compatible: `brcm,bcm2711-dma`
    - driver: `downstream CONFIG_DMA_BCM2835`
  - `rpivid-local-intc`
    - compatible: `raspberrypi,rpivid-local-intc`
    - driver: `downstream rpivid-mem`
  - `vp9-decoder`
    - compatible: `raspberrypi,rpivid-vp9-decoder`
    - driver: `downstream rpivid-mem`
  - `hevc-decoder`
    - compatible: `raspberrypi,rpivid-hevc-decoder`
    - driver: `downstream rpivid-mem`
  - `h264-decoder`
    - compatible: `raspberrypi,rpivid-h264-decoder`
    - driver: `downstream rpivid-mem`
- `soc`
  - compatible: `simple-bus`
  - `watchdog`
    - compatible: `brcm,bcm2835-pm`
    - driver: `CONFIG_ARCH_BCM2835`
    - controls power domains, reset lines, and a watchdog timer
    - takes `BCM2835_CLOCK_{V3D,PERI_IMAGE,H264,ISP}` as inputs and outputs
      `BCM2835_POWER_DOMAIN_*`
  - `power`
    - compatible: `raspberrypi,bcm2835-power`
    - driver: `CONFIG_RASPBERRYPI_POWER`
    - outputs `RPI_POWER_DOMAIN_*`
  - `cprman`
    - compatible: `brcm,bcm2711-cprman`
    - driver: `CONFIG_CLK_BCM2835`
    - takes external oscillator as input and outputs `BCM2835_CLOCK_*` clocks
  - `aux`
    - compatible: `brcm,bcm2835-aux`
    - driver: `CONFIG_CLK_BCM2835`
    - takes `BCM2835_CLOCK_VPU` as input and outputs `BCM2835_AUX_CLOCK_*`
  - `clock`
    - compatible: `brcm,brcm2711-dvp`
    - driver: `CONFIG_CLK_BCM2711_DVP`
  - `interrupt-controller`
    - compatible: `arm,gic-400`
    - driver: `CONFIG_ARM_GIC`
  - `local_intc`
    - compatible: `brcm,bcm2836-l1-intc`
    - driver: `CONFIG_ARCH_BCM2835`
    - per-cpu interrupt controller for timer, PMU, IPIs
  - `mailbox`
    - compatible: `brcm,bcm2835-mbox`
    - driver: `CONFIG_BCM2835_MBOX`
  - `mailbox`
    - compatible: `brcm,bcm2711-vchiq`
    - driver: `downstream CONFIG_BCM2835_VCHIQ`
    - `bcm2835_audio`
      - compatible: `brcm,bcm2835-audio`
      - driver: `downstream?`
      - status: disabled
  - `firmware`
    - compatible: `raspberrypi,bcm2835-firmware`
    - driver: `CONFIG_RASPBERRYPI_FIRMWARE`
    - `clocks`
      - compatible: `raspberrypi,firmware-clocks`
      - driver: `CONFIG_CLK_RASPBERRYPI`
    - `gpio`
      - compatible: `raspberrypi,firmware-gpio`
      - driver: `CONFIG_GPIO_RASPBERRYPI_EXP`
  - `timer`
    - compatible: `brcm,bcm2835-system-timer`
    - driver: `CONFIG_BCM2835_TIMER`
    - a freerunning counter plus 4 timer channels
  - `dma`
    - compatible: `brcm,bcm2835-dma`
    - driver: `CONFIG_DMA_BCM2835`
    - up to 16 channels
  - `rng`
    - compatible: `brcm,bcm2711-rng200`
    - driver: `CONFIG_HW_RANDOM_IPROC_RNG200`
  - `usb`
    - compatible: `brcm,bcm2708-usb`
    - driver: `downstream?`
    - status: disabled
  - `axiperf`
    - compatible: `brcm,bcm2835-axiperf`
    - driver: `downstream?`
    - status: disabled
    - AXI performance monitor
  - `avs-monitor`
    - compatible: `brcm,bcm2711-avs-monitor`
    - driver: `none?`
    - `thermal`
      - compatible: `brcm,bcm2711-thermal`
      - driver: `CONFIG_BCM2711_THERMAL`
  - `serial`
    - compatible: `arm,pl011`
    - driver: `CONFIG_SERIAL_AMBA_PL011`
    - this is `/dev/ttyAMA0`
    - `bluetooth`
      - compatible: `brcm,bcm43438-bt`
      - driver: `CONFIG_BT_HCIUART_BCM`
  - `serial`
    - compatible: `brcm,bcm2835-aux-uart`
    - driver: `CONFIG_SERIAL_8250_BCM2835AUX`
    - this is `/dev/ttyS0`
    - `bluetooth`
      - compatible: `brcm,bcm43438-bt`
      - driver: `CONFIG_BT_HCIUART_BCM`
      - status: disabled
  - `serial` * 4
    - compatible: `arm,pl011`
    - driver: `CONFIG_SERIAL_AMBA_PL011`
    - status: disabled
  - `csi` * 2
    - compatible: `brcm,bcm2835-unicam`
    - driver: `downstream?`
    - status: disabled
  - `mmcnr`
    - compatible: `brcm,bcm2835-mmc`
    - driver: `downstream mmc-bcm2835`
    - this connects to BCM4345/6 over SDIO
  - `mmc`
    - compatible: `brcm,bcm2835-mmc`
    - driver: `downstream?`
    - status: disabled
  - `mmc`
    - compatible: `brcm,bcm2835-sdhost`
    - driver: `CONFIG_MMC_BCM2835`
    - status: disabled
  - `gpio`
    - compatible: `brcm,bcm2711-gpio`
    - driver: `CONFIG_PINCTRL_BCM2835`
    - followed by a lot of pin definitions
    - a combined GPIO controller, GPIO interrupt controller, and a
      pinmux/control device
    - takes `BCM2835_CLOCK_PWM` as input
  - `gpiomem`
    - compatible: `brcm,bcm2835-gpiomem`
    - driver: `downstream gpiomem-bcm2835`
  - `i2c0mux`
    - compatible: `i2c-mux-pinctrl`
    - driver: `CONFIG_I2C_MUX_PINCTRL`
    - status: disabled
  - `i2c` * 2
    - compatible: `brcm,bcm2711-hdmi-i2c`
    - driver: `CONFIG_I2C_BRCMSTB`
  - `i2c` * 6
    - compatible: `brcm,bcm2711-i2c`
    - driver: `CONFIG_I2C_BCM2835`
    - status: disabled
  - `spi` * 2
    - compatible: `brcm,bcm2835-aux-spi`
    - driver: `CONFIG_SPI_BCM2835AUX`
    - status: one ok and one disabled
    - takes `BCM2835_AUX_CLOCK_SPI[1-2]` as inputs
  - `spi` * 5
    - compatible: `brcm,bcm2835-spi`
    - driver: `CONFIG_SPI_BCM2835`
    - status: disabled
    - `spidev`, for one of the `spi`
      - compatible: `spidev`
      - driver: `downstream?`
  - `i2s`
    - compatible: `brcm,bcm2835-i2s`
    - driver: `CONFIG_SND_BCM2835_SOC_I2S`
    - status: disabled
    - takes `BCM2835_CLOCK_PCM` as input
  - `smi`
    - compatible: `brcm,bcm2835-smi`
    - driver: `downstream?`
    - status: disabled
    - Secondary Memory Interface
  - `pwm` * 2
    - compatible: `brcm,bcm2835-pwm`
    - driver: `CONFIG_PWM_BCM2835`
    - status: disabled
    - takes `BCM2835_CLOCK_PWM` as input
    - controls power to motors, etc.
  - `vcsm`
    - compatible: `raspberrypi,bcm2835-vcsm`
    - driver: `downstream bcm2835-vcsm`
  - `fb`
    - compatible: `brcm,bcm2708-fb`
    - driver: `downstream?`
    - status: disabled
  - `firmwarekms`
    - compatible: `raspberrypi,rpi-firmware-kms-2711`
    - driver: `downstream CONFIG_DRM_VC4`
    - status: disabled
    - vc4 is modified to do KMS over firmware dismanx instead of registers
  - `txp`
    - compatible: `brcm,bcm2835-txp`
    - driver: `CONFIG_DRM_VC4`
  - `hvs`
    - compatible: `brcm,bcm2835-hvs`
    - driver: `CONFIG_DRM_VC4`
  - `pixelvalve` * 5
    - compatible: `brcm,bcm2711-pixelvalve.`
    - driver: `CONFIG_DRM_VC4`
  - `vec`
    - compatible: `brcm,bcm2835-vec`
    - driver: `CONFIG_DRM_VC4`
    - status: disabled
  - `hdmi` * 2
    - compatible: `brcm,bcm2711-hdmi.`
    - driver: `CONFIG_DRM_VC4`
  - `dsi` * 2
    - compatible: `brcm,bcm2835-dsi.`
    - driver: `CONFIG_DRM_VC4`
    - status: disabled
    - takes `BCM2835_PLLD_DSI1` and `BCM2835_CLOCK_DSI1[EP]` as inputs
  - `dpi`
    - compatible: `brcm,bcm2835-dpi`
    - driver: `CONFIG_DRM_VC4`
    - status: disabled
    - takes `BCM2835_CLOCK_{VPU,DPI}` as inputs

## `/sys/bus`

- `amba` bus
  - `fe201000.serial` driven by `uart-pl011`
- `cec` bus
  - HDMI consumer electronics control
  - `cec[0-1]`
- `clockevents` bus
  - `clockevent[0-3]`
  - `broadcast`
- `clocksource` bus
  - `clocksource0`
- `cpu` bus
  - `cpu[0-3]`
- `event_source` bus
  - `armv7_cortex_a15`
  - `kprobe`
  - `tracepoint`
  - etc
- `gpio` bus
  - `gpiochip[0-1]`
- `i2c` bus
  - `i2c-[11-12]`
- `mdio_bus` bus
  - `unimac-mdio`
- `media` bus
  - `media[0-1]`
- `mmc` bus
  - `mmc[0-1]:*`
- `nvmem` bus
- `pci` bus
  - one driven by `pcieport`
  - another driven by `xhci_hcd`
- `pci_express` bus
  - one driven by `pcie_pme` for power management
  - another driven by `aer` for advanced error reporting
- `platform` bus
  - `3e2fd000.framebuffer` driven by `simple-framebuffer`
  - `arm-pmu` driven by `armv7-pmu`
  - `bcm2835-camera`
  - `bcm2835-power` driven by `bcm2835-power`
  - `bcm2835-wdt` driven by `bcm2835-wdt`
  - `cpufreq-dt` driven by `cpufreq-dt`
  - `fd500000.pcie` driven by `brcm-pcie`
  - `fd580000.ethernet` driven by `bcmgenet`
  - `fd5d2000.avs-monitor:thermal` driven by `bcm2711_thermal`
  - `fe003000.timer`
  - `fe004000.txp` driven by `vc4_txp`
  - `fe007000.dma` and `fe007b00.dma` driven by `bcm2835-dma`
  - `fe00b880.mailbox` driven by `bcm2835-mbox`
  - `fe100000.watchdog` driven by `bcm2835-pm`
  - `fe101000.cprman` driven by `bcm2835-clk`
  - `fe104000.rng` driven by `iproc-rng200`
  - `fe200000.gpio` driven by `pinctrl-bcm2835`
  - `fe200000.gpiomem` driven by `gpiomem-bcm2835`
  - `fe215000.aux` driven by `bcm2835-aux-clk`
  - `fe300000.mmcnr` driven by `mmc-bcm2835`
  - `fe340000.emmc2` driven by `sdhci-iproc`
  - `fe400000.hvs` driven by `vc4_hvs`
  - `feb00000.hevc-decoder`, `feb10000.rpivid-local-intc`,
    `feb20000.h264-decoder`, and `feb30000.vp9-decoder` driven by `rpivid-mem`
  - `fec00000.v3d` driven by `v3d`
  - `fe206000.pixelvalve`, `fe207000.pixelvalve`, `fe20a000.pixelvalve`,
    `fe216000.pixelvalve` and `fec12000.pixelvalve` driven by `vc4_crtc`
  - `fef00000.clock` driven by `brcm2711-dvp`
  - `fef00700.hdmi` and `fef05700.hdmi` driven by `vc4_hdmi`
  - `fef04500.i2c` and `fef09500.i2c` driven by `brcmstb-i2c`
  - `ff800000.local_intc`
  - `fixedregulator_3v3`, `fixedregulator_5v0`, and `sd_vcc_reg` driven by
    `reg-fixed-voltage`
  - `gpu` driven by `vc4-drm`
  - `kgdboc` driven by `kgdboc`
  - `leds` driven by `leds-gpio`
  - `phy`
  - `raspberrypi-cpufreq` driven by `raspberrypi-cpufreq`
  - `reg-dummy` driven by `reg-dummy`
  - `regulatory.0`
  - `sd_io_1v8_reg` driven by `gpio-regulator`
  - `snd-soc-dummy` driven by `snd-soc-dummy`
  - `timer`
  - `unimac-mdio.-19` driven by `unimac-mdio`
- `platform` bus (talk to VC firmware)
  - `bcm2835-codec` driven by `bcm2835-codec`
  - `bcm2835-isp` driven by `bcm2835-isp`
  - `fe00b840.mailbox` driven by `bcm2835_vchiq`
  - `raspberrypi-hwmon` driven by `raspberrypi-hwmon`
  - `soc:firmware` driven by `raspberrypi-firmware`
  - `soc:firmware:clocks` driven by `raspberrypi-clk`
  - `soc:firmware:gpio` driven by `raspberrypi-exp-gpio`
  - `soc:power` driven by `raspberrypi-power`
  - `soc:vcsm` driven by `bcm2835-vcsm`
  - `vcsm-cma` driven by `vcsm-cma`
- `sdio` bus
  - `mmc1:0001:[1-3]` driven by `brcmfmac`
- `serial` bus
  - `serial0-0` driven by `hci_uart_bcm`
- `usb` bus
  - some devices driven by `hub` or `usb`

## More Devices

- lsusb
  - one 5G/s root hub driven by `xhci_hcd/4p`
  - one 480M/s root hub driven by `xhci_hcd/1p`
- lspci
  - one Broadcom pci bridge driven by `pcieport`
  - one VIA VL805 usb 3.0 controller driven by `xhci_hcd`
- 6 uarts
  - <https://www.raspberrypi.org/documentation/configuration/uart.md>
  - uart 0 is `/dev/ttyAMA0` driven by `amba-pl011`
  - uart 1 is `/dev/ttyS0`
  - uart [2-5] is disabled by default
- one BT/WIFI controller
  - the WIFI part is driven by `brcmfmac`
  - add `dtparam=krnbt=on` to `config.txt` to enable the BT part in DT and
    allow `btbcm` to drive it
  - otherwise, use `btattach` in userspace to attach `/dev/ttyAMA0`
- one GPU
  - managed by propriertary driver by default
  - add `dtoverlay=vc4-kms-v3d-pi4` to `config.txt` to enable the device in DT
    and allow `vc4` and `v3d` to drive it
  - `vc4` drives the display part
  - `v3d` drives the 3D part
- no RTC
  - get time from NTP

## Unused drivers

- BCM2835 VCHIQ firmware service
  - compatible: brcm,bcm283[5-6]-vchiq
  - driver: `CONFIG_BCM2835_VCHIQ`
  - can potentially support brcm,bcm2711-vchiq?
- BCM2835 Thermal Sensor
  - compatible: brcm,bcm283[5-7]-thermal
  - driver: `CONFIG_BCM2835_THERMAL`
  - replaced by `CONFIG_BCM2711_THERMAL`
- BCM2835 Random number generator
  - compatible: brcm,bcm2835-rng
  - driver: `CONFIG_HW_RANDOM_BCM2835`
  - replaced by `CONFIG_HW_RANDOM_IPROC_RNG200`
- BCM2835 Top-Level ("ARMCTRL") Interrupt Controller
  - up to 72 interrupt sources
  - compatible: brcm,bcm283[5-6]-armctrl-ic
  - driver: `CONFIG_ARCH_BCM2835`
  - replaced by `CONFIG_ARM_GIC`?
- Raspberry Pi VideoCore firmware reset
  - compatible: raspberrypi,firmware-reset
  - driver: `CONFIG_RESET_RASPBERRYPI`
  - ???
- BCM2835 VideoCore V3D GPU
  - compatible: brcm,bcm2835-v3d
  - driver: `CONFIG_DRM_VC4`
  - replaced by `CONFIG_DRM_V3D`

## PXE

- update EEPROM and `BOOT_ORDER`
  - <https://github.com/raspberrypi/rpi-eeprom>
  - extrat config from fw file (or eeprom)
  - edit `BOOT_ORDER`
  - apply config to fw file (or eeprom)
  - flash
- eeprom looks for "Raspberry Pi Boot" PXE boot option
  - with dnsmasq, specify `--pxe-service=0,"Raspberry Pi Boot"`
- it fetches these files with empty `config.txt`
  - <https://github.com/raspberrypi/firmware>
  - `config.txt`
  - `start4.elf`
  - `fixup4.dat`
  - `bcm2711-rpi-4-b.dtb`
  - `overlays/overlay_map.dtb`
  - `cmdline.txt`
  - `kernel7l.img`

## Partitioning

- see <../distros/disk.md>
- need to update to the latest eeprom for GPT support
