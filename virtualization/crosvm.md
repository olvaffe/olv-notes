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
  - `cargo build`
  - `cargo run`
- run
  - `./target/debug/crosvm run \
       -c 4 \
       -m 4096 \
       -r <disk-image> \
       -p <kernel-params> \
       -i <initramfs-image> \
       <kernel-image>`
  - using a COW2 image with Arch pre-installed in second partition
    - disk-image: arch.qcow2
    - kernel-params: debug root=/dev/vda2
    - initramfs-image: (extract from arch.qcow2)
    - kernel-image: (extract from arch.qcow2)
  - the disk image is mounted read-only
    - replace `-r` by `--rwroot`
