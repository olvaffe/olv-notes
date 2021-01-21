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
      - `PciRoot` does not implement `PciDevice`
      - but it owns `PciRootConfiguration` which implements `PciDevice`
    - for each device,
      - an eventfd is created and `ioctl(KVM_IRQFD)`
      - if the device can be notified (each vq of each device can),
        `ioctl(KVM_IOEVENTFD)`
      - if the device is jailed, wrap it in `ProxyDevice`
        - this is where the device process is forked
        - this creates yet another socket for communication
      - add the io ranges to `mmio_bus`
  - create `PciConfigIo` which implements BusDevice
    - to handle PIC config address via port 0xcf8 and 0xcfc
    - it holds a pointer to pci root
  - `setup_io_bus`
    - create an `io_bus` and adds `PciConfigIo` to it
    - also legacy devices such as rtc, i8042
  - `setup_serial_devices` adds COM[0-3] to `io_bus`
  - `setup_acpi_devices` adds acpi pm device to `io_bus`
  - manually set up mptable, smbios, and acpi tables (DSDT/FACP/MADT/XSDT)
  - load kernel image, cmdline, initramfs into GuestMemory
  - `configure_system`
    - set up boot header for kernel
    - add e820 entries
- `run_control`
  - spawns one thread for each vcpu and runs inside `run_vcpu` forever
  - the main thread enters a loop forever to poll and handle fds

## vcpu

- main thread
  - `run_control` calls `run_vcpu` to spawn a vcpu thread
- each vcpu thread calls `runnable_vcpu` first to create a RunnableVcpu
  - `create_cpu` calls `ioctl(KVM_CREATE_VCPU)` to get a vcpu fd
  - mmap first `ioctl(KVM_GET_VCPU_MMAP_SIZE)` bytes of vcpu fd
    - the region holds a `struct kvm_run` that can be used to communiate wit
      the vcpu
  - `configure_vcpu` sets up vcpu cpuid, fpu, segments, and registers
    - it also follows the kernel boot protocol if BIOS is not used
- each vcpu thread then enters a loop calling `ioctl(KVM_RUN)`
  - the thread spends most of the time inside `ioctl(KVM_RUN)` running guest
    code
  - on vmexit, it may handle the request in kernel and resume to run guest
    code; but sometimes, it needs to return from the ioctl and let the
    userspace handle the request
- common exit reasons
  - `VcpuExit::IoIn(port, size)` reads from `io_bus` at `port`
    - writes to `kvm_run::io::data_offset` where vcpu can see it
  - `VcpuExit::IoOut(port, size, data)` writes to `io_bus` at `port`
  - `VcpuExit::MmioRead(addr, size)` reads from `mmio_bus` at `addr`
    - writes to `kvm_run::mmio::data)`
  - `VcpuExit::MmioWrite(addr, size, data)` writes to `mmio_bus` at `addr`

## Device I/O

- there are two `devices::bus::Bus`: `io_bus` and `mmio_bus`
  - they handle pio and mmio respectively
  - reads or writes are handled by `devices::bus::Bus::{write,read}`
  - every device on the bus implements `BusDevice`
  - `addr` is first translated to `(dev, addr)` and then BusDevice's `write`
    or `read` is called
- on `io_bus`, there is `PciConfigIo` which implements `BusDevice`
  - used by kernel's `arch/x86/pci/early.c` and `arch/x86/pci/direct.c`
    - when the kernel calls `pci_read_config_*`, it reaches `direct.c` and
      uses this device ultimately
  - when read at 0xcfc, it calls `pci_root::PciConfigIo::config_space_read` to
    read from `self.config_address`.  This is done by translating
    `self.config_address` to pci `(bus, dev, func) and `reg`.
- when sandboxed, the device is ProxyDevice that imeplements `BusDevice`
  - `config_register_write` sends a one-way `Command::WriteConfig`
  - `config_register_read` sends a synchronous `Command::ReadConfig`
  - `write` sends a one-way `Command::Write`
  - `read` sends a synchronous `Command::Read`
- when not sandboxed, or when in the device process, the device is
  `VirtioPciDevice` (or `XhciController`, `PciRootConfiguration`, etc.) that
  implements `BusDevice`.  For `VirtioPciDevice`,
  - `config_register_write` calls the device's `PciConfiguration`'s
    `write_reg` (unless it is MSI reg)
  - `config_register_read` calls `PciConfiguration`'s `read_reg`
  - `write` is MMIO and calls `write_bar`
  - `read` is MMIO and calls `read_bar`
  - virtio write/read MMIO is commonly used to access
    - the common `VirtioPciCommonConfig`
    - the common `MsixConfig`
    - the device-specific config such as `virtio_gpu_config`
- virtio vq notify is handled by kernel
  - the device is set up with `ioctl(KVM_IOEVENTFD)`
  - when the guest writes to the registered mmio address, the vmexit is
    handled by kernel and the eventfd is signaled

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
- `ProxyDevice::new` wraps a `BusDevice` and forks
  - the main process does device i/o normally.  The proxy device transparently
    talk to the device process
  - the device process loops in `child_proc` indefinitely to handle device i/o
    requests from the main process

## virtio

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
- all virtio devices have `PciClassCode::Other` and
  `PciVirtioSubclass::NonTransitionalBase`
  - when the device type is `TYPE_GPU`, it should at least be
    `PciClassCode::DisplayController` for Xorg primary gpu auto selection to
    work

## virtio-gpu

- the main thread waits inside `devices::proxy::child_proc`
- the gpu thread is spawned from `activate`
  - `--gpu backend=3d` sets the backend kind to `BackendKind::Virtio3D`
  - the gpu thread calls `BackendKind::build`
    - this calls `DisplayBackend::build` first which calls `XOpenDisplay`
    - it then calls `virtio_3d_backend::Virtio3DBackend::build` which
      initializes a `Renderer` which calls `virgl_renderer_init`
  - it then enters `devices::virtio::gpu::Worker::run` and never leaves
- the gpu thread mainly
  - wakes up with event `Token::CtrlQueue` ready when the guest notifies
    - guest notifies -> vmexit -> host kernel signals eventfd
  - for each descritopr in the queue, `process_queue`
    - calls `process_descriptor`
    - calls `process_gpu_command`
- guest can use `VIRTIO_GPU_CMD_RESOURCE_CREATE_3D` to allocate a
  virglrenderer resource
  - `devices::virtio::gpu::Frontend::process_gpu_command`
  - `devices::virtio::gpu::virtio_gpu::VirtioGpu::resource_create_3d`
  - `rutabaga_gfx::rutabaga_core::Rutabaga::resource_create_3d`
  - `virgl_renderer_resource_create`
  - note that,
    - `Rutabaga::resource_create_3d` actually calls both
      `virgl_renderer_resource_create` and
      `virgl_renderer_resource_export_blob`
      - export can fail
    - `VirtioGpu::resource_create_3d` also calls `virgl_renderer_execute` to
      query
      - this is optional and can fail
- guest can also use `VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB` to allocate a
  virglrenderer resource
  - `devices::virtio::gpu::virtio_gpu::VirtioGpu::resource_create_3d`
  - `rutabaga_gfx::rutabaga_core::Rutabaga::resource_create_3d`
  - `virgl_renderer_resource_create_blob`
  - similar to 3D resources, they create/export/query
- `VIRTIO_GPU_CMD_RESOURCE_MAP_BLOB` is used to map a blob resource into guest
  - `devices::virtio::gpu::virtio_gpu::VirtioGpu::resource_map_blob`
  - it asks rutabaga to `virgl_renderer_resource_get_map_info` and retreives
    the already exported fd, if there is an exported fd
    - it then calls `VmMemoryRequest::RegisterFdAtPciBarOffset` to map the fd
      into the guest
  - otherwise, it calls `virgl_renderer_resource_map` to map without an fd
    (in-process only)

## ResourceBridge

- ResourceBridge is a connection between virtio-gpu and virtio-wl (and others)
- guest kernel uses `VIRTIO_WL_CTRL_VFD_SEND_KIND_VIRTGPU` to send a guest
  dma-buf to host wayland
  - guest virtio-wl driver first calls `virtio_dma_buf_get_uuid` to get the
    uuid of a dma-buf
    - for virtio-gpu dma-buf, uuid is from
      `VIRTIO_GPU_CMD_RESOURCE_ASSIGN_UUID` thus is determined by crosvm
  - guest virtio-wl driver sends the uuid with the wayland traffic to host
- crosvm sees `VIRTIO_WL_CTRL_VFD_SEND_KIND_VIRTGPU` and...
  - uses `ResourceBridge` to send `ResourceRequest::GetBuffer` from virtio-wl
    to virtio-gpu
  - virtio-gpu wakes up to `process_resource_bridge` and `export_resource`
    - the resource must be already exported
    - `virgl_renderer_execute` is called to query
  - virtio-wl nows gets the host dmabuf
    - the stack is `Worker::run`, `WlState::execute`, `WlState::send`, and
      `WlState::get_info`

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

## Crates

- `sys_util` crate
  - `syslog`
    - `syslog::init` is called in `crosvm::crosvm_main` to initialize
    - `syslog::log` logs to syslog and optionally stderr and fd
      - `echo_stderr` to enable stderr (default on)
      - `echo_file` to set custom fd (default off)
    - macros: error!, warn!, info!, debug!, log!
      - all call `syslog::log`
