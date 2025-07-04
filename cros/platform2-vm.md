Platform2 vm
============

## Crostini Overview

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
    - calls `TerminaVm::Create`
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

- <https://chromium.googlesource.com/chromiumos/platform2/+/refs/heads/master/vm_tools/>
- to get a shell, run `vmc start termina` in host
- log at host `/run/daemon-store/crosvm/<hash>/log/<random>.log`
- kernel
  - `console=hvc0 earlycon=uart8250,io,0x3f8 root=/dev/pmem0 ro`
  - `/dev/pmem0` is raw ext4 image `vm_rootfs.img`
  - maitred runs as pid 1
- maitred
  - `Init::Setup()`
    - mounts `/proc`, `/sys`, `/tmp`, `/mnt/external`, `/run`, `/dev/shm`,
      `/dev/pts`, `/var`, `/sys/fs/cgroup`, 
    - starts `vm_syslog` and `vshd`
  - starts grpc server
    - controlled by host `maitred_client`
  - listens to gRPCs from `vm_concierge`
    - mainly fromn `Service::StartVm` in `vm_concierge`
    - `ServiceImpl::ConfigureNetwork` sets up guest network
    - `ServiceImpl::Mount` mounts specified images
      - `/dev/vda`, which is ro raw ext4 image `vm_tools.img`, to
      	`/opt/google/cros-containers`
    - `ServiceImpl::Mount9P` mounts 9p
    - `ServiceImpl::StartTermina`
      - mount `/dev/vdb`, which is rw raw btrfs image, to `/mnt/stateful`
      - spawns `crash_reporter`
      - spawns `lxcfs`
      - spawns `tremplin`
      - spawns `ndproxyd` and `mcastd`
        - <https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/patchpanel>
- tremplin runs inside the VM and provides an gRPC interface for LXD container management
  - source code at `src/platform/tremplin`
  - it starts lxd and dnsmasq
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
- `lxc profile show default`
  - `raw.idmap: ...` remaps uid/gid for unprivileged container
  - `security.nesting: "true"` allows nested containers
    - it mount `/sys` and `/proc` rw
    - it creates `/dev/.lxc`
    - no real security impact with unprivileged container

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

## vmc

- launching crostini remotely
  - `/usr/local/autotest/bin/autologin.py`
  - `dlcservice_util --id=termina-dlc --install`
  - `vmc start --enable-gpu termina`
  - `vmc container termina penguin https://storage.googleapis.com/cros-containers/%d debian/bookworm`
  - `vmc container termina penguin`
  - `vmc stop termina`
- borealis
  - `dlcservice_util --id=borealis-dlc --install`
  - `vmc start --enable-vulkan --no-start-lxd --enable-big-gl --dlc-id=borealis-dlc --extra-disk=disk.img borealis`

## Arch Linux

- enable crostini in settings
- `Ctrl-Alt-T` to open crosh
- `vsh termina` to enter vm
- `lxc remote add canonical https://images.lxd.canonical.com --protocol=simplestreams`
  to add an alternative image server
  - because the `lxc` lost access to <https://images.linuxcontainers.org/>
- `lxc launch canonical:archlinux arch` to create and launch an instance for
  archlinux
- `lxc list` to confirm the instance state
  - it must be running and have an ipv4 address
- `lxc exec arch -- bash`
  - if there is no ipv4 address, `systemctl edit systemd-networkd` to add
    - `[Service]`
    - `BindReadOnlyPaths=/sys`
  - `sudo pacman -S git`
  - `git clone https://aur.archlinux.org/cros-container-guest-tools-git.git`
  - `makepkg -s`
  - `sudo pacman -U cros-container-guest-tools-git-r470.63de46b4-1-any.pkg.tar.zst`
- `cros-container-guest-tools-git` package
  - it starts `garcon` on the user session
- container profile and config
  - `lxc profile show default`
    - it bind-mounts `/opt/google/cros-containers` into the container
  - `lxc config show arch`
- `vmc container termina arch`
  - `user_id_hash_to_username` returns the username based on the account email
  - `container_create` sends `CreateLxdContainerRequest` to `cicerone` to
    create the container on demand
    - tremplin inside the termina vm creates the lxd container
  - `container_setup_user` sends `SetUpLxdContainerUserRequest` to `cicerone`
    to setup the user account in the container on demand
    - tremplin inside the termina vm does the real work
    - it creates the required users and groups
    - it creates `/var/lib/systemd/linger/<user>` to start the user session on
      boot
  - `container_start` sends `StartLxdContainerRequest` to `cicerone` to start
    the container on demand
    - tremplin inside the termina vm starts the lxd container
  - `vsh_exec_container` executes `vsh` to send `LaunchVshdRequest` to
    `cicerone`
    - garcon inside the container does the real work and runs
      `/opt/google/cros-containers/bin/vshd`
    - vsh is a remote shell and is vsock-based
