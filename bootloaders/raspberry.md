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
