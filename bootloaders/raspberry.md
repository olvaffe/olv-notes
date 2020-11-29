Raspberry Pi
============

## SoC

- three main processors with 3 different instruction sets
  - a VPU, dual-core dual-issue 16-way SIMD for system managements, codecs, etc.
  - QPUs, for 3D works
  - an ARM CPU, to run applications

## Boot Process on RPi4

- When powered on, VPU executes stage1 bootloader on SoC ROM
  - it initializes eMMC controller
  - it executes stage2 from recovery.bin on sdcard, or from EEPROM
  - CPU is reset and SDRAM is disabled
- VPU executes stage2 bootloader (normally on EEPROM)
  - it initializes more of the system, including SDRAM
  - config is written to EEPROM as well
  - it loads stage3 from `startup4.elf` on sdcard or usb to SDRAM
- VPU executes stage3 bootloader (`starup4.elf`) on SDRAM
  - it parses `config.txt`
  - it loads various AP (ARM CPU) images
  - ARM CPU is out of reset and starts running
  - stage3 bootloader is actually a full RTOS based on ThreadX
- ARM executes some initializations, armstub, and kernel
  - <https://github.com/raspberrypi/tools/tree/master/armstubs>
- open source firmwares
  - stage2/3 replacement, <https://github.com/christinaa/rpi-open-firmware>
    - it initializes the system similar to stage2
    - it initializes ARM to run an embedded chainloader
    - the chainloader (re-)initializes eMMC controller and loads linux kernel
    - most hw blocks do not work
  - EFI, <https://github.com/pftf/RPi4>
    - it is built from official EDK2 and backed by ARM and VMWare
    - it instructs stage3 (via `config.txt`) to use EDK2 as armstub
    - it can chainload any EFI bootloader (grub2, systemd-boot,
      linux kernel itself)

## PXE

- update EEPROM and `BOOT_ORDER`
  - <https://github.com/raspberrypi/rpi-eeprom>
  - extrat config from fw file (or eeprom)
  - edit `BOOT_ORDER`
  - apply confi to fw file (or eeprom)
  - flash
- it looks for "Raspberry Pi Boot" PXE boot option
- these files are fetched
  - <https://github.com/raspberrypi/firmware>
  - config.txt
  - start4.elf
  - fixup4.dat
  - bcm2711-rpi-4-b.dtb
  - overlays/overlay_map.dtb
  - cmdline.txt
  - kernel7l.img

## Devices

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
- one audio device
  - driven by `snd_bcm2835`

## More Devices

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

## Partitioning

- partitioning
  - fdisk and `g` to use GPT
    - need to update to the latest eeprom
  - partition 1: 260M, ESP
  - partition 2: `32*1000-260`M, Linux
- filesystems
  - partition 1: `mkfs.fat -F32`
  - partition 2: `mkfs.btrfs`
- prepare btrfs
  - mount partition 2
  - `mkdir roots homes`
  - `btrfs subvolume create roots/current`
  - `btrfs subvolume create homes/current`
  - `btrfs subvolume set-default roots/current`
  - umount
- btrfs
  - mount partition 2 again with `subvol=roots/current`
  - `mkdir boot home`
  - mount partition 1 to boot
  - mount partition 2 to home with `subvol=homes/current`
