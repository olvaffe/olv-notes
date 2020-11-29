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
