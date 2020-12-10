Crostini
========

## Overview

- it all starts with `vmc start --enable-gpu termina`
  - ask chrome to download `cros-termina` component (`chrome://components`)
  - ask debugd to `StartVmConcierge`
  - ask `vm_concierge` to `CreateDiskImage`
  - ask `vm_concierge` to `StartVm`
  - run `vsh` to get the shell to termina
- to start debian container, `vmc container termina penguin`
  - ask `vm_cicerone` to `CreateLxdContainer`, which downloads
    `debian/stretch` container from
    `https://storage.googleapis.com/cros-containers` by default
  - ask `vm_cicerone` to `StartLxdContainer`
  - ask `vm_cicerone` to `SetUpLxdContainerUser`
  - run `vsh` to get the shell to the container

## Disk Images

- When asked, Chrome downloads `cros-termina` component to
  `/home/chronos/cros-components/cros-termina/<version>`
  - the component is shareable with all users
  - the component contains `image.ext4` raw ext4 image
  - the disk image is loop-mounted at
    `/run/imageloader/cros-termina/<version>`
  - the disk image contains
    - `vm_kernel` is the kernel
    - `vm_rootfs.img` is a raw ext4 image
    - `vm_tools.img` is a raw ext4 image
- When asked, `vm_concierge` creates a disk image under
  `/home/root/<user-hash>/crosvm`
  - the disk image is private to the user
  - the disk image is raw and is formatted to btrfs
  - the path is bind-mounted to
    `/run/daemon-store/crosvm/<hash>`
- crosvm is started with
  - `--pmem-device vm_rootfs.img` and mounted as ro rootfs by guest
  - `--disk vm_tools.img` and mounted as ro `/opt/google/cros-containers` by
    guest
  - `--rwdisk <created-disk-image>` and mounted as rw `/mnt/stateful` by guest

## Host

- `vmc` source code is at `src/platform2/vm_tools/crostini_client`
- `cros-termina` component is installed to `/run/imageloader/cros-termina`
- `vm_concierge` is started by `start vm_concierge` on demand during `vmc start`
  - `CreateDiskImage` creates a disk image under `/home/root/<id>/crosvm` by
    default
  - the disk image has a max size of 90% of the free space, as requested by
    `vmc`
  - `StartVm` runs `crosvm` with runtime directory at `/run/vm`
  - it calls `vm->Mount`, which requests `/sbin/init` inside VM to mount `/dev/vd[bcdef...]`
    - normally, `/dev/vdb` is backed by an image file under
      `/home/root/<id>/crosvm` on host and is mounted at `/mnt/stateful` in
      VM
  - it calls `vm->Mount9P`, which request `/sbin/init` inside VM to mount `/mnt/shared`
- `vm_cicerone` talks to `tremplin` inside the VM to manage LXD containers
- `vsh` is a remote shell over `virtio-vsock`, a `AF_VSOCK` socket that
  connects guest/host
  - there is `vshd` runs inside the VM

## Termina

- to get a shell, run `vmc start termina` in host
- tremplin runs inside the VM and provides an gRPC interface for LXD container management
  - source code at `src/platform/tremplin`
  - when requested to `CreateLxdContainer`, it downloads the container from
    the LXD image server
  - when requested to `StartLxdContainer`, it starts the specified container
  - when requested to `SetUpUser`, it updates `/etc/passwd` and `/etc/groups`
    in the container
- `vm_syslog` is the syslog daemon running inside the VM and relays the logs
  to `vmlog_forwarder`, which in turn relays the logs to host syslog daemon

## Penguin

- to get a shell, run `vmc container termina penguin` in host
  - or, run `lxc exec penguin -- /bin/bash` in termina
- Xwayland is run
- sommelier is a wayland compositor that forwards the traffic to the host
  wayland compositor via `/dev/wl0`

## Processes

- `imageloader` to load disk images
- `crosdns`
- `vmlog_forwarder`
- `seneschal` manages 9P servers
- `vm_concierge`
- `crosvm run --cpus 4 --mem 5901 --root /run/imageloader/... --tap-fd 16 --cid 3`
  - `--socket /run/vm/vm.Ucaif1/crosvm.sock`
  - `--wayland-sock /run/chrome/wayland-0`
  - `--wayland-dmabuf`
  - `--gpu`
