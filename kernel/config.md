Kernel Config
=============

## Prepare

- `make alldefconfig`
- `make menuconfig`
- select `Processor type and features`
  - select `Symmetric multi-processing support`
- select `Enable loadable module support`
- select `Networking support`
- select `Device Drivers`
  - select `PCI support`
  - select `USB support`
    - select `Support for Host-side USB`
  - select `LED support`
    - select `LED Class Support`
- select `Kernel hacking`
  - select `Kernel debugging`

## General setup

- deselect `Automatically append version information to the version string`
- deselect `Support for paging of anonymous memory (swap)`
- select `System V IPC`
- select `POSIX Message Queues`
- select `Timers subsystem`
  - select `Timer tick handling (Idle dynticks system (tickless idle))`
  - select `High Resolution Timer Support`
- select `Preemption Model (Voluntary Kernel Preemption (Desktop))`
- select `CPU/Task time and stats accounting`
  - select `BSD Process Accounting`
- select `Kernel .config support`
  - select `Enable access to .config through /proc/config.gz`
- select `Control Group support`
- select `Initial RAM filesystem and RAM disk (initramfs/initrd) support`

## Processor type and features

- deselect `Enable MPS table`
- deselect `Support for extended (non-PC) x86 platforms`
- select `Processor family (Core 2/newer Xeon)`
- deselect `AMD MCE features`
- select `EFI runtime service support`
  - select `EFI stub support`
- select `Timer frequency (1000 HZ)`

## Power management and ACPI options

- select `Cpuidle Driver for Intel Processors`

## Binary Emulations

- select `IA32 Emulation`

## Virtualization

- select `Kernel-based Virtual Machine (KVM) support`
  - select `KVM for Intel processors support`

## Enable loadable module support

- select `Module unloading`

## Executable file formats

- select `Kernel support for MISC binaries`

## Networking support

- select `Networking options`
  - select `Packet socket`
  - select `Unix domain sockets`
  - select `TCP/IP networking`
- select `Bluetooth subsystem support`
- select `Wireless`
  - select `cfg80211 - wireless configuration API`
  - select `Generic IEEE 802.11 Networking Stack (mac80211)`
- select `RF switch subsystem support`

## Device Drivers

- select `PCI support`
  - select `PCI Express Port Bus support`
  - select `Message Signaled Interrupts (MSI and MSI-X)`
- select `Generic Driver Options`
  - select `Maintain a devtmpfs filesystem to mount at /dev`
- select `Block devices`
  - select `Loopback device support`
- select `NVME Support`
  - select `NVM Express block device`
- select `SCSI device support`
  - select `SCSI device support`
  - select `SCSI disk support`
  - deselect `SCSI low-level drivers`
- select `Serial ATA and Parallel ATA drivers (libata)`
  - select `AHCI SATA support`
- select `Network device support`
  - select `Ethernet driver support`
    - deselect all but the desired drivers
  - select `USB Network Adapters`
    - deselect all but the desired drivers
  - select `Wireless LAN`
    - deselect all but the desired drivers
- select `Input device support`
  - deselect `Support for memoryless force-feedback devices`
  - select `Event interface`
  - select `Synaptics RMI4 bus support`
  - select `RMI4 SMB Support`
- select `Character devices`
  - deselect `Legacy (BSD) PTY support`
  - select `Hardware Random Number Generator Core support`
    - deselect all but the desired drivers
  - deselect `/dev/port character device`
- select `Multimedia support`
  - select `Cameras/video grabbers support`
  - select `Media USB Adapters`
    - select `USB Video Class (UVC)`
    - deselect `GSPCA based webcams`
- select `Graphics support`
  - select `Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)`
  - select `Intel 8xx/9xx/G3x/G4x/HD Graphics`
  - select `Backlight & LCD device support`
    - deselect `Generic (aka Sharp Corgi) Backlight Driver`
- select `Sound card support`
  - select `Advanced Linux Sound Architecture`
    - select `HD-Audio`
      - select `HD Audio PCI`
      - select `Build Realtek HD-audio codec support`
- select `HID support`
  - select `Special HID drivers`
    - deselect all but the desired drivers, such as
    - select `Synaptics RMI4 device support`
- select `USB support`
  - select `xHCI HCD (USB 3.0) support`
  - select `EHCI HCD (USB 2.0) support`
  - select `USB Printer support`
  - select `USB Mass Storage support`
- select `Real Time Clock`
- deselect `Virtio drivers`
- select `X86 Platform Specific Device Drivers`
  - select `Dell Systems Management Base Driver`
  - select `Dell SMBIOS driver`
  - select `Dell Laptop Extras`
- deselect `IOMMU Hardware Support`

## File systems

- select `The Extended 4 (ext4) filesystem`
- select `Btrfs filesystem support`
- deselect `Dnotify support`
- select `FUSE (Filesystem in Userspace) support`
- select `CD-ROM/DVD Filesystems`
  - select `ISO 9660 CDROM file system support`
  - select `Microsoft Joliet CDROM extensions`
- select `DOS/FAT/NT Filesystems`
  - select `VFAT (Windows-95) fs support`
  - select `Enable FAT UTF-8 option by default`
- deselect `Miscellaneous filesystems`
- deselect `Network File Systems`
- select `Native language support`
  - select `Codepage 437 (United States, Canada)`
  - select `NLS ISO 8859-1  (Latin 1; Western European Languages)`
  - select `NLS UTF-8`

## Kernel hacking

- select `printk and dmesg options`
  - select `Show timing information on printks`
- select Tracers
  - select `Trace process context switches and events`

## Tips

- Unable to boot?
  - Built-in script (#!) support for init scripts
  - Built-in devtmpfs and unix socket support for udev
  - Built-in graphics driver to see console
  - Built-in keyboard driver to interact
  - Built-in scsi/ata/fs drivers to mount

## KVM

- Host
  - select `Virtualization`
    - select `Host kernel accelerator for virtio net`
    - select `VHOST_SCSI TCM fabric driver`
    - select `vhost virtio-vsock driver`
  - select `Networking support`
    - select `Networking options`
      - select `Virtual Socket protocol`
  - select `Device Drivers`
    - select `Generic Target Core Mod (TCM) and ConfigFS Infrastructure`
- Guest
  - disable virtualization, host drivers, EFI, etc.
  - select `Device Drivers`
    - select `Network device support`
      - select `Virtio network driver`
    - select `Character devices`
      - select `Serial drivers`
        - select `8250/16550 and compatible serial support`
        - select `Console on 8250/16550 and compatible serial port`
    - select `Graphics support`
      - select `Virtio GPU driver`
    - select `Virtio drivers`
      - select `PCI driver for virtio devices`
      - select `Virtio input driver`

## Out-of-tree drivers

- Broadcom wireless
  - `pacman -S broadcom-wl-dkms`
  - `dkms autoinstall`
