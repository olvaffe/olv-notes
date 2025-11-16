Windows
=======

## Installation

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
