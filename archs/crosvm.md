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
- network
  - run crosvm as root and specify
    - `--host_ip 192.168.0.1`
    - `--netmask 255.255.255.0`
    - `--mac 12:34:56:78:9a:bc`
  - in the guest
    - `ip link set enp0s4 up`
    - `ip addr add 192.168.0.2/24 dev enp0s4`
    - `ip route add default via 192.168.0.2`
  - in the host
    - set up NAT
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

## Startup Flow

- entrypoint `crosvm::main` in `src/main.rs`
  - `crosvm::run_vm` handles the `run` subcommand
    - it creates a default config and parses the args to update the config
  - `crosvm::platform::run_config` runs the config
  - `crosvm::platform::run_vm` creates sockets for devices and builds the vm
  - `crosvm::platform::run_control`
- `build_vm` is called by `run_vm` before `run_control` to build the vm
  - `setup_memory` creates memfd
    - there are usually 2 to 3 ranges
      - BIOS (optional)
      - [.. 3.25GB] (`MEM_32BIT_GAP_SIZE`)
      - [4GB ..]
    - GuestMemory creates memfd
  - `create_vm` points to `create_kvm`
    - it opens `/dev/kvm`
    - it calls `ioctl(KVM_CREATE_VM)` to get vm fd
      - the vm has no cpu nor memory yet
    - `ioctl(KVM_SET_USER_MEMORY_REGION)` for each memory region
      - the vm has memory now but still no cpu
  - `create_irq_chip` points to `create_kvm_kernel_irq_chip`
    - `ioctl(KVM_CREATE_IRQCHIP)`
      - in-kernel virtual ioapic, virtual pic (two PICs, nested), 
    - `ioctl(KVM_CREATE_PIT2)`
      - in-kernel PIT timer
    - there is `create_kvm_split_irq_chip` to emulate the devices in userspace
      instead, but is experimental
  - `create_devices` points to `crosvm::platform::create_devices`
    - `create_virtio_devices` create virtio devices
      - note that each virtio device
        - is put in minijails unless `--disable-sandbox`
        - has at least one socket for communication
      - for each `--serial`, `create_console_device`
      - for each `--disk`, `create_block_device`
      - for each `--pmem-device`, `create_pmem_device`
        - this does `ioctl(KVM_SET_USER_MEMORY_REGION)`
      - always `create_rng_device`
      - for each `--evdev`, `create_vinput_device`
      - always `create_balloon_device`
      - if `--host_ip` or `--tap-fd`, `create_net_device`
      - if `--enable-gpu`, `create_gpu_device`
      - for each `--shared-dir`, `create_fs_device`
    - for each virtio device,
      - another socket is created for MSI request/response
      - VirtioPciDevice is created to wrap the virtio device and the MSI
      	socket
    - a `XhciController` is created
  - `generate_pci_root`
    - this creates a pci root and adds all devices to it
    - for each device,
      - an eventfd is created and `ioctl(KVM_IRQFD)`
      - if the device can be notified (each vq of each device can),
        `ioctl(KVM_IOEVENTFD)`
      - if the device is jailed, wrap it in `ProxyDevice`
        - this is where the device process is forked
        - this creates yet another socket for communication
        - there are only four operations handled by
            `devices::proxy::child_proc`
          - for pci device, they are read bar, write bar, read config, write
            config
  - `setup_io_bus`
    - create an io bus and adds pci root to it
    - also legacy devices such as rtc, i8042
  - `setup_serial_devices` adds COM[0-3] to io bus
  - `setup_acpi_devices`
    - adds acpi pm device to io bus
  - manually set up mptable, smbios, and acpi tables (DSDT/FACP/MADT/XSDT)
  - load kernel image, cmdline, initramfs into GuestMemory
  - `configure_system`
    - set up boot header for kernel
    - add e820 entries
- `run_control`
  - spawns one thread for each vcpu and runs inside `run_vcpu` forever
  - the main thread enters a loop forever to poll and handle fds

## Sandboxing

- unless `--disable-sandbox` is specified, sandboxing is enabled
- it uses minijail crate to
  - set up namespaces
  - set up seccomp
- `simple_jail`
  - checks `/var/empty` and will use it for `pivot_root`
  - parses `.policy` under `/usr/share/policy/crosvm`
  - uses pids/user/vfs/net namespaces, with all user caps removed
    - some devices don't use `simple_jail` to keep user caps
  - sets up the minijail config

## virtio

- all virtio devices have `PciClassCode::Other` and
  `PciVirtioSubclass::NonTransitionalBase`
- when the device type is `TYPE_GPU`, it should at least be
  `PciClassCode::DisplayController` for Xorg primary gpu auto selection to
  work

## virtio-wl

- when a guest app calls X, sommelier does Y
  - `wl_shm_create_pool` -> `sl_shm_create_host_pool`
  - `wl_shm_pool_create_buffer` -> `sl_host_shm_pool_create_host_buffer`
  - `wl_drm_create_prime_buffer` -> `sl_drm_create_prime_buffer`
  - `wl_compositor_create_surface` -> `sl_compositor_create_host_surface`
  - `wl_surface_attach` -> `sl_host_surface_attach`
    - this creates a buffer in the host and maps it into the guest
    - the exact host buffer depends on shm driver
- guest can use `VIRTIO_WL_CMD_VFD_NEW_DMABUF` to allocate a host dmabuf and
  map it in the guest
- in virtio-wl process, it calls
  - `WlState::new_dmabuf`
  - `WlVfd::dmabuf`
  - `VmRequester::request(VmMemoryRequest::AllocateAndRegisterGpuMemory)`
- in crosvm process, it calls
  - `SystemAllocator::gpu_memory_allocator`
  - `GpuBufferDevice::allocate`
  - `Device::create_buffer`
  - `gbm_bo_create`

## virtio-gpu

- guest can use `VIRTIO_GPU_CMD_RESOURCE_CREATE_3D` to allocate a
  virglrenderer resource
- in virtio-gpu process, it calls
  - `Virtio3DBackend::resource_create_3d`
  - `Renderer::resource_create`
  - `virgl_renderer_resource_create`

## virtio-device

- VM devices are created in `create_devices`
  - To create a virtio-device, a VirtioDevice is created first.  It is then
    wrapped in a `VirtioPciDevice`.  The device can be optionally put in a
    jail.
  - Then in `generate_pci_root`, the pci device is wrapped in a `ProxyDevice`,
    if jail is enabled.  It forks a child process, puts it in jail, and enters
    `child_proc` indefinitely
- When there is a MMIO that should be handled by a virtio-device, the vcpu
  exits with `VcpuExit::MmioWrite` or `VcpuExit::MmioRead`
  - the vcpu thread calls `write()` method on the MMIO bus
  - it gets translated to a `write()` method on the device that is mapped at
    the address
  - this is a `ProxyDevice`.  The write gets translated to a `Command::Write`
  - `child_proc` of the device process handles the command and translates that
    to a `write_bar`
- When a virtio-device MMIO write indicates that the guest driver is ready, it
  `activate`s the device
  - the virtio-device normally spawns a thread to process virtqueues
- When a virtio-device MMIO write indicates a vq notification, it is delivered
  through eventfd instead
  - thanks to `KVM_IOEVENTFD`
  - the vq thread polls on the eventfd, gets notified, and processes the
    buffers
- when a virtio-device compeltes a vq buffer, it writes to an eventfd to
  generate an IRQ
  - thanks to `KVM_IRQFD`
- when a virtio-device needs the main process to do something, such as
  injecting pages into the guest, it sends a `VmMemoryRequest`

## Crates

- `sys_util` crate
  - `syslog`
    - `syslog::init` is called in `crosvm::crosvm_main` to initialize
    - `syslog::log` logs to syslog and optionally stderr and fd
      - `echo_stderr` to enable stderr (default on)
      - `echo_file` to set custom fd (default off)
    - macros: error!, warn!, info!, debug!, log!
      - all call `syslog::log`
