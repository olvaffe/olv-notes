crosvm
======

## Build and Run

- download source code
  - `mkdir crosvm`
  - `cd crosvm`
  - `repo init -g crosvm -u https://chromium.googlesource.com/chromiumos/manifest.git \
               --repo-url=https://chromium.googlesource.com/external/repo.git`
  - `repo sync`
- install dependencies
  - `sudo apt install libcap-dev libfdt-dev \
                      libgbm-dev libvirglrenderer-dev \
                      libwayland-bin libwayland-dev wayland-protocols \
		      protobuf-compiler`
- build
  - `cd src/platform/crosvm`
  - `cargo build --features "gpu x"`
  - `cargo run`
- run
  - `./target/debug/crosvm run \
       --disable-sandbox \
       -c 4 \
       -m 4096 \
       --gpu width=800,height=600 \
       --display-window-keyboard \
       --display-window-mouse \
       --rwdisk <disk-image> \
       -p <kernel-params> \
       -i <initramfs-image> \
       <kernel-image>`
  - for sandboxing, remove `--disable-sandbox` and make sure to
    - `sudo mkdir /var/empty`
    - `sudo mkdir /usr/share/policy`
    - `sudo ln -s /path/to/crosvm/seccomp/x86_64 /usr/share/policy/crosvm`
    - seems to conflict with `--gpu`, need to debug
- to install Arch Linux
  - create new disk image
    - `./target/debug/crosvm create_qcow2 arch.qcow2 20000000000`
  - download ISO image and extract kernel/initramfs
    - `7z x <iso-image> arch/boot/x86_64`
  - start crosvm with the ISO image as the second disk
    - `--rwdisk arch.qcow2`
    - `--disk <iso-image>`
    - `-p archisodevice=/dev/vdb1`
    - `-i arch/boot/x86_64/initramfs-linux.img`
    - `arch/boot/x86_64/vmlinuz-linux`
  - install Arch Linux to `/dev/vda`
    - follow the installation guide
    - remember to extract the installed kernel/initramfs for the host

## virtio

- all virtio devices have `PciClassCode::Other` and
  `PciVirtioSubclass::NonTransitionalBase`
- when the device type is `TYPE_GPU`, it should at least be
  `PciClassCode::DisplayController` for Xorg primary gpu auto selection to
  work
