Microsoft
=========

## Windows Installation

- use media creation tool to create bootable usb
  - dd iso to usb does not work
    - it boots but cannot find nvme to install to
- Or, create Bootable USB manually
  - download disk image (iso)
  - partition usb with an esp partition
  - `mkfs.fat -F32 /dev/sda1`
  - mount iso and usb
  - `rsync -rt --exclude sources/install.wim iso/ usb/`
  - `wimsplit iso/sources/install.wim usb/sources/install.swm 1024`
- during installation,
  - if the disk is already partitioned, it refuses to install to any existing
    partition
    - must delete a partition and install to the unallocated space
    - this creates 3 partitions from the unallocated space
      - `Microsoft reserved`: win requires each disk to have a MSR partition
      - `Microsoft basic data`: this is drive C and is bitlocked
      - `Windows recovery environment`: this is WinRE for recovery
  - network is required
    - might need to download drivers from vendor/motherboard website and unzip
      them to (second?) usb

## Windows Partitioning

- <https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions>
- the device can have multiple disks
  - both GPT and MBR are supported
  - the disk containing the Windows partition must use GPT
- types of partitions
  - System partition (ESP)
  - Microsoft reserved partition (MSR)
  - OEM partitions
  - Windows partition
  - Recovery tools partition
  - Data partitions
- ESP
  - must have at least 1 ESP
  - minimum size is 100MB
  - must be FAT32
    - FAT32 requires a minimum size of 260MB on 4KB-sector disks (and a
      minimum size of 36MB on 512B-sector disks)
      - because it requires a minimum of 65527 clusters plus some for metadata
    - `mkfs.fat -F 32 -S 4096` warns when the partition size is
      `65695 * 4096` or less
    - `mkfs.fat -F 32 -S 4096 -a -f 1 -h 0 -R 2 -s 1` warns when the
      partition size is `65590 * 4096` or less
    - `mkfs.fat -F 32` warns when the partition size is `66591 * 512` or less
- MSR
  - 16MB
  - each GPT disk should have a MSR partition
- OEM partitions
  - must locate before Windows/Resovery/Data partitions
  - must set preserved bit
- Windows partition
  - minimum size is 20GB
    - for win11, 128GB
      - <https://www.microsoft.com/en-us/windows/windows-11-specifications>
      - 64GB for installation
      - double that for updates
  - must be NTFS
  - must be on a GPT disk
- Recovery tools partition
  - minimum size is 300MB
  - should follow Windows partition immediately
- Data partitions
  - not recommended
  - should follow Recovery tool partition immediately

## DOS History

- early microcomputers have tapes or have no permanent storage
- as disks become affordable, disk operating systems (DOS) become popular
- non-x86 dos
  - CBM DOS, Commodore
  - AmigaDOS, Amiga, 1985
  - Apple DOS, Apple, 1978-1983
  - ProDOS, Apple, 1983-1993
  - TR-DOS, ZX Spectrum, 1984-1986
- x86 dos
  - CP/M, Digital Research, 1974
  - 86-DOS, Seattle Computer, 1980
  - MS-DOS, Microsoft, 1981
  - PC-DOS, Microsoft / IBM, 1981
  - DR-DOS, Digital Research, 1988
  - FreeDOS, open source, 1994
