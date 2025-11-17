Kernel VFIO
===========

## VFIO

- a device driver that exposes the device to userspace for direct access
- e.g., vfio-pci is a universal PCI device driver
  - given a PCI device, we need to ask the kernel to unbind the current driver
  - after ubind, we ask the kernel to bind it to vfio-pci driver
- device passthrough
  - VM is in userspace, unprivileged
  - VMM can passthrough the device to VM, and let the VM drives it
- SR-IOV
  - the physical device is referred to as PF, Physical Function
  - its driver can ask it to allocate a virtual device, VF
  - we want vfio-pci rather than the real driver to drive VF
  - this way, we can pass through the VF to the VM
  - VM can drive the VF using the real driver
- how about devices that don't support SR-IOV?
  - the real driver for the physical device can add support for mdev, or
    mediated devices
  - the driver can allocate mdev
  - vfio-mdev can drive the mdev
  - Done.
  - it is the real driver's job to make mdevs appears as independent devices
- it seems people want this now
  - the real driver supports mdev and emulates it as a virtio device
  - the benefit is that the guest only needs virtio drivers

