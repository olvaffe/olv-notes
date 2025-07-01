Windows
=======

## Installation

- Create Bootable USB
  - download disk image (iso)
  - partition usb with an esp partition
  - `mkfs.fat -F32 /dev/sda1`
  - mount iso and usb
  - `rsync -rt --exclude sources/install.wim iso/ usb/`
  - `wimsplit iso/sources/install.wim usb/sources/install.swm 1024`
