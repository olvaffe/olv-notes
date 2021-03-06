Kernel Config
=============

## Flow

- config
  - `make alldefconfig` to generate config
    - or `make defconfig` to use arch defaults, which include common device
      drivers for the arch
    - or download a pre-made config and `make olddefconfig`
      - such as
        <https://raw.githubusercontent.com/raspberrypi/linux/rpi-5.10.y/arch/arm64/configs/bcm2711_defconfig>
  - `make menuconfig` to edit generated config
    - to discover non-discoverable devices,
      - `dtc -I fs /sys/firmware/devicetree/base` and look for `compatible`
        property.  Kernel uses the property to bind drivers.
      - `find /sys/devices/LNXSYSTM:00 -name modalias`.  Kernel uses modalias
        to bind drivers.  (It is constructed from _HID and _CID of the ACPI
        devices btw)
    - to discover discoverable devices,
      - lspci
      - lsusb
- build
  - `make` to build kernel image, modules, and dtbs
- install
  - `make install` to invoke `installkernel`
  - `make modules_install` to install to `INSTALL_MOD_PATH`
- cross compile
  - get cross compiler
    - `crossbuild-essential-arm64` on Debian-like
    - `aarch64-linux-gnu-gcc` on Arch-like
  - set variables
    - `ARCH=arm64`
    - `CROSS_COMPILE=aarch64-linux-gnu-`
  - manual install
    - copy `arch/arm64/boot/Image` for ARM64
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

## Tips

- Unable to boot?
  - must built-in for initramfs to work
    - script (#!) support for init scripts
    - devtmpfs and unix socket support for udev
  - to debug
    - Built-in EFI or Simple framebuffer to see console
    - Built-in keyboard driver to interact
    - Built-in scsi/ata/fs drivers to mount
- No `/dev/dri`?
  - `echo 0x1ff > /sys/module/drm/parameters/debug`
  - `[drm:vc5_hdmi_init_resources [vc4]] ERROR Failed to get HDMI state machine clock`
    - make sure `vc4` is loaded after `raspberrypi-clk`
  - `[drm:vc4_hdmi_bind [vc4]] Failed to get ddc i2c adapter by node
    - make sure `vc4` is loaded after `i2c-brcmstb`

## Config: First Pass

- quick first pass to reveal more options
- select `Enable loadable module support`
- select `Networking support`
- select `Device Drivers`
  - select `PCI support`
  - select `USB support`
    - select `Support for Host-side USB`
- select `Kernel hacking`
  - select `Kernel debugging`
- for x86
  - select `Processor type and features`
    - select `Symmetric multi-processing support`
- for Raspberry Pi
  - select `Platform selection`
    - select `Broadcom BCM2835 family`
  - select `Device Drivers`
    - select `Mailbox Hardware Support`
      - select `BCM2835 Mailbox`
  - select `Firmware Drivers`
    - select `Raspberry Pi Firmware Driver`
  - select `Device Drivers`
    - select `Character devices`
      - select `Serial device bus` (for bluetooth)
    - select `I2C support`
      - select `I2C support`
    - select `SPI support`
    - select `Sound card support`
      - select `Advanced Linux Sound Architecture`
        - select `ALSA for SoC audio support`
    - select `Common Clock Framework`
      - select `Raspberry Pi firmware based clock support`

## Config: General Setup

- select `General setup`
  - deselect `Automatically append version information to the version string`
  - select `System V IPC`
  - select `POSIX Message Queues`
  - select `Timers subsystem`
    - select `Timer tick handling (Idle dynticks system (tickless idle))`
    - select `High Resolution Timer Support`
  - select `Preemption Model (Voluntary Kernel Preemption (Desktop))`
  - select `Kernel .config support`
    - select `Enable access to .config through /proc/config.gz`
  - select `Control Group support`
  - select `Automatic process group scheduling`
  - select `Initial RAM filesystem and RAM disk (initramfs/initrd) support`
  - select `Kernel Performance Events And Counters`
    - select `Kernel performance events and counters`
- select `Enable loadable module support`
  - select `Module unloading`
- select `Executable file formats`
  - select `Kernel support for MISC binaries`
- select `Memory Management options`
  - select `Transparent Hugepage Support`
  - select `Transparent Hugepage Support sysfs defaults (madvice)`
  - select `Contiguous Memory Allocator` if needed
    - needed by some DRM drivers such as vc4
- select `Networking support`
  - select `Networking options`
    - select `Packet socket`
    - select `Unix domain sockets`
    - select `TCP/IP networking`
    - select `Network packet filtering framework (Netfilter)`
      - select `Core Netfilter Configuration`
        - select `Netfilter connection tracking support`
        - select `Network Address Translation support`
        - select `Netfilter nf_tables support`
        - select `Netfilter nf_tables masquerade support`
        - select `Netfilter nf_tables nat module`
      - select `IP: Netfilter Configuration`
        - select `IPv4 nf_tables support`
  - select `Bluetooth subsystem support`
    - select `Bluetooth device drivers` and enable desired drivers
      - select `HCI USB driver`
      - select `HCI UART driver` and `Broadcom protocol support`
  - select `Wireless`
    - select `cfg80211 - wireless configuration API`
    - select `Generic IEEE 802.11 Networking Stack (mac80211)`
  - select `RF switch subsystem support`
- select `File systems`
  - select `Btrfs filesystem support`
  - select `Btrfs POSIX Access Control Lists`
  - deselect `Dnotify support`
  - select `Kernel automounter support (supports v3, v4 and v5)` for systemd
  - select `FUSE (Filesystem in Userspace) support`
  - select `DOS/FAT/NT Filesystems`
    - select `VFAT (Windows-95) fs support`
    - select `Enable FAT UTF-8 option by default`
    - select `exFAT filesystem support`
  - select `Pseudo filesystems`
    - select `Tmpfs virtual memory file system support (former shm fs)`
    - select `Tmpfs POSIX Access Control Lists`
    - select `Tmpfs extended attributes`
  - deselect `Miscellaneous filesystems`
  - deselect `Network File Systems`
  - select `Native language support`
    - select `Codepage 437 (United States, Canada)`
    - select `NLS ISO 8859-1  (Latin 1; Western European Languages)`
    - select `NLS UTF-8`
- select `Library routines`
  - select `DMA Contiguous Memory Allocator` if needed
    - needed by some DRM drivers such as vc4
- select `Kernel hacking`
  - select `printk and dmesg options`
    - select `Show timing information on printks`
  - select `Generic Kernel Debugging Instruments`
    - select `Debug Filesystem`
  - select `Tracers`
    - select `Kernel Function Tracer`
    - select `Trace syscalls`

## Config: x86

- select `Processor type and features`
  - deselect `Enable MPS table`
  - deselect `Support for extended (non-PC) x86 platforms`
  - if Intel
    - select `Intel Low Power Subsystem Support`
    - select `Processor family (Core 2/newer Xeon)`
    - deselect `AMD MCE features`
  - if AMD
    - select `AMD ACPI2Platform devices support`
    - select `Processor family (Opteron/Athlon64/Hammer/K8)`
    - deselect `Intel MCE features`
    - deselect `Intel microcode loading support`
    - select `AMD microcode loading support`
  - select `EFI runtime service support`
    - select `EFI stub support`
  - select `Timer frequency (300 HZ)`
- select `Power management and ACPI options`
  - select `CPU Frequency scaling`
    - select `ACPI Processor P-States driver` if AMD
  - select `Cpuidle Driver for Intel Processors` if Intel
- select `Binary Emulations`
  - select `IA32 Emulation`
- deselect `Virtualization`
- select `General architecture-dependent options`
  - select `Provide system calls for 32-bit time_t`
- select `Device Drivers`
  - select `PCI support`
    - select `PCI Express Port Bus support`
    - select `Message Signaled Interrupts (MSI and MSI-X)`
  - select `Generic Driver Options`
    - select `Maintain a devtmpfs filesystem to mount at /dev` for systemd
    - select `Automount devtmpfs at /dev, after the kernel mounted the rootfs`
  - select `Block devices`
    - select `Loopback device support`
  - select `NVME Support` if have one
    - select `NVM Express block device`
  - select `Misc devices`
    - select `Realtek PCI-E card reader` if have one
  - select `SCSI device support` if have SATA or USB drives
    - deselect `legacy /proc/scsi/ support`
    - select `SCSI device support`
    - select `SCSI disk support`
    - deselect `SCSI low-level drivers`
  - select `Serial ATA and Parallel ATA drivers (libata)` if have one
    - select `AHCI SATA support`
    - deselect `ATA SFF support (for legacy IDE and PATA)`
  - select `Network device support`
    - select `Ethernet driver support` if have one
    - select `USB Network Adapters` if have one
    - select `Wireless LAN` if have one
  - select `Input device support`
    - select `Event interface`
  - select `Character devices`
    - deselect `Legacy (BSD) PTY support`
    - select `Hardware Random Number Generator Core support`
      - deselect all but the desired drivers
    - select `TPM Hardware Support`
      - select `TPM 2.0 CRB Interface` if have one
  - select `I2C support`
    - select `I2C Hardware Bus support`
      - select `Intel PIIX4 and compatible (ATI/AMD/Serverworks/Broadcom/SMSC)` if AMD
      - select `Synopsys DesignWare Platform` for Intel/AMD CPUs
  - select `Pin controllers`
    - select `AMD GPIO pin control` if AMD
  - select `Hardware Monitoring support`
    - select `AMD Family 10h+ temperature sensor` if AMD
    - select `Intel Core/Core2/Atom temperature sensor` if Intel
  - select `Thermal drivers`
    - select `Intel thermal drivers`
      - select `ACPI INT340X thermal drivers`
        - select `ACPI INT340X thermal drivers`
      - select `Intel PCH Thermal Reporting Driver`
  - select `Watchdog Timer Support`
    - select `AMD/ATI SP5100 TCO Timer/Watchdog` if AMD
    - select `Intel TCO Timer/Watchdog` if Intel
  - select `Multimedia support`
    - select `Media device types`
      - select `Cameras/video grabbers support`
    - select `Media drivers`
      - select `Media USB Adapters`
        - select `USB Video Class (UVC)`
        - deselect `GSPCA based webcams`
  - select `Graphics support`
    - select `Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)`
    - select `Intel 8xx/9xx/G3x/G4x/HD Graphics`
    - select `Frame buffer Devices`
      - select `Support for frame buffer devices`
        - select `EFI-based Framebuffer Support`
  - select `Sound card support`
    - select `Advanced Linux Sound Architecture`
      - deselect `Support old ALSA API`
      - select `HD-Audio`
        - select `HD Audio PCI`
        - select `Build Realtek HD-audio codec support` if have one
        - select `Build HDMI/DisplayPort HD-audio codec support` if have one
      - select `ALSA for SoC audio support`
        - select `AMD Audio Coprocessor - Renoir support` if AMD
  - select `HID support`
    - select `Special HID drivers`
      - deselect all but the desired drivers, such as
      - select `HID Multitouch panels`
      - select `Wacom Intuos/Graphire tablet support (USB)`
      - select `HID Sensors framework support` for sensors
    - select `I2C HID support`
      - select `HID over I2C transport layer` for I2C touchpads
    - select `AMD SFH HID Support`
      - select `AMD Sensor Fusion Hub` if AMD
  - select `USB support`
    - select `xHCI HCD (USB 3.0) support`
    - select `EHCI HCD (USB 2.0) support`
    - select `USB Printer support`
    - select `USB Mass Storage support`
    - select `USB Type-C Support`
      - select `USB Type-C Connector System Software Interface driver`
      - select `UCSI ACPI Interface Driver`
  - select `MMC/SD/SDIO card support`
    - select `Realtek PCI-E SD/MMC Card Interface Driver`
  - select `LED support`
    - select `LED Class Support`
  - select `Real Time Clock`
  - select `DMA Engine support`
    - select `Synopsys DesignWare AHB DMA platform driver` for Intel LPSS
  - deselect `Virtio drivers`
  - deselect `VHOST drivers`
  - select `X86 Platform Specific Device Drivers`
    - select `Dell Systems Management Base Driver` if Dell
    - select `Dell SMBIOS driver` if Dell
    - select `Dell Laptop Extras` if Dell
    - select `Lenovo IdeaPad Laptop Extras` if Lenovo IdeaPad
  - deselect `IOMMU Hardware Support`
  - select `Industrial I/O support`
    - select `Accelerometers`
      - select `HID Accelerometers 3D` if accelerometers (tablets, 2-in-1s)
  - select `Generic powercap sysfs driver`
    - select `Intel RAPL Support via MSR Interface` for both Intel and AMD
- select `Cryptographic API`
  - if iwd,
    - select `User-space interface for hash algorithms`
    - and many others as indicated by the log
  - select `Hardware crypto devices`
    - select `Support for AMD Secure Processor`

## Config: arm64

- select `Kernel Features`
  - select `Timer frequency (300 HZ)`
  - select `Kernel support for 32-bit EL0`
    - select `Emulate deprecated/obsolete ARMv8 instructions`
      - select all
- select `Boot options`
  - deselect `UEFI runtime support`
- select `CPU Power Management`
  - select `CPU Idle`
    - select `CPU idle PM support`
    - select `ARM CPU Idle Drivers`
      - select `Generic ARM/ARM64 CPU idle Driver`
  - select `CPU Frequency scaling`
    - select `CPU Frequency scaling`
    - select `Generic DT based cpufreq driver`
    - select `Raspberry Pi cpufreq support` if needed
- select `Device Drivers` for Raspberry Pi
  - select `PCI support`
    - select `PCI controller drivers`
      - select `Broadcom Brcmstb PCIe host controller`
  - select `Generic Driver Options`
    - select `Maintain a devtmpfs filesystem to mount at /dev` for systemd
    - select `Automount devtmpfs at /dev, after the kernel mounted the rootfs`
  - select `Network device support`
    - select `Ethernet driver support`
      - select `Broadcom GENET internal MAC support`
    - select `Wireless LAN`
      - select `Broadcom FullMAC WLAN driver`
  - select `Input device support`
    - select `Event interface`
    - deselect `Keyboards`
    - deselect `Mice`
  - select `I2C support`
    - select `I2C Hardware Bus support`
      - select `Broadcom BCM2835 I2C controller`
  - select `SPI support`
    - select `BCM2835 SPI controller`
    - select `BCM2835 SPI auxiliary controller`
  - select `Hardware Monitoring support`
    - select `Raspberry Pi voltage monitor`
  - select `Thermal drivers`
    - select `Broadcom thermal drivers`
      - select `Thermal sensors on bcm2835 SoC`
  - select `Watchdog Timer Support`
    - select `Broadcom BCM2835 hardware watchdog`
  - select `Voltage and Current Regulator Support`
    - select `Fixed voltage regulator support`
    - select `GPIO regulator support`
  - select `Graphics support`
    - select `Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)`
    - select `Broadcom VC4 Graphics`
    - select `Frame buffer Devices`
      - select `Support for frame buffer devices`
        - select `Simple framebuffer support`
  - select `USB support`
    - select `xHCI HCD (USB 3.0) support`
  - select `MMC/SD/SDIO card support`
    - select `Secure Digital Host Controller Interface support`
    - select `SDHCI platform and OF driver helper`
    - select `SDHCI support for the BCM2835 & iProc SD/MMC Controller`
  - select `LED support`
    - select `LED Class Support`
  - select `Real Time Clock`
  - select `DMA Engine support`
    - select `BCM2835 DMA engine support`
  - select `Staging drivers`
    - select `Broadcom VideoCore support`
      - select all
  - select `SOC (System On Chip) specific Drivers`
    - select `Broadcom SoC drivers`
      - select `Raspberry Pi power domain driver`
  - select `Pulse-Width Modulation (PWM) Support`
    - select `BCM2835 PWM support`
  - select `Reset Controller Support`
    - select `Raspberry Pi 4 Firmware Reset Driver`

## Config: Containers

- select `General setup`
  - select `Namespaces support`
    - select `User namespace`
- select `Networking support`
  - select `Networking options`
    - select `802.1d Ethernet Bridging`

## Config: KVM Host

- select `Virtualization`
  - select `Kernel-based Virtual Machine (KVM) support`
    - select `KVM for Intel processors support`
- select `Networking support`
  - select `Networking options`
    - select `Virtual Socket protocol`
- select `Device Drivers`
  - select `VHOST drivers`
    - select `Host kernel accelerator for virtio net`
    - select `vhost virtio-vsock driver`
  - select `IOMMU Hardware Support`
    - select `Support for Intel IOMMU using DMA Remapping Devices`
    - select `Support for Interrupt Remapping`

## Config: KVM Guest

- select `Processor type and features`
  - select `Support x2apic`
  - select `Linux guest support`
    - select `Enable paravirtualization code`
    - select `Paravirtualization layer for spinlocks`
- select `Networking support`
  - select `Networking options`
    - select `Virtual Socket protocol`
    - select `virtio transport for Virtual Sockets`
- select `Device Drivers`
  - select `Block devices`
    - select `Virtio block driver`
  - select `Network device support`
    - select `Virtio network driver`
  - select `Character devices`
    - select `Serial drivers`
      - select `8250/16550 and compatible serial support`
      - select `Console on 8250/16550 and compatible serial port`
    - select `Virtio console`
  - select `Graphics support`
    - select `Virtio GPU driver`
  - select `Virtio drivers`
    - select `PCI driver for virtio devices`
    - select `Virtio input driver`
- deselect virtualization, host drivers, EFI, etc.
