crosvm
======

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
