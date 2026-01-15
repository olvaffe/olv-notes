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
