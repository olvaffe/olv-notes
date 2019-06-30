# virtio

## Core

* the core registers a `virtio` bus
  * the bus matches devices with drivers
  * each virtio driver calls `register_virtio_driver` to register itself to
    the bus
  * the virtio-pci driver discovers PCI virtio devices, allocates a
    `virtio_pci_device` which is a subclass of `virtio_device`, and
    `register_virtio_device` the virtio device to the bus
* the core registers a virtio-pci driver for PCI devices
  * it matches all `PCI_VENDOR_ID_REDHAT_QUMRANET` PCI devices
  * for each matched `pci_dev`, it allocates a `virtio_pci_device` which is a
    child of the `pci_dev`
    * it calls `pci_enable_device` to enable the PCI device
    * it calls `virtio_pci_modern_probe` to initialize the virtio device
      * or falls back to `virtio_pci_legacy_probe`
      * maps `common`, `device`, and `notify_base` regions for MMIO
      * initializes `virtio_config_ops` to `virtio_pci_config_ops`, for use by
      	the virtio driver
      * initializes `setup_vq`, `del_vq`, and `config_vector` callbacks to
      	abstract the differences between legacy/modern PCI virtio devices
    * it calls `register_virtio_device` to register the device to the virtio
      bus
* each virtio device has a `virtio_config_ops` to config the device
  * in `register_virtio_device`
    * the device is always `->reset` first when registered
    * then `VIRTIO_CONFIG_S_ACKNOWLEDGE` is set to the status register
  * in `virtio_dev_probe`
    * set `VIRTIO_CONFIG_S_DRIVER` to the status register
    * decide features
      * get device features from the device feature register
      * get driver features from the driver feature table
      * the usable features are those set in both masks
      * `>finalize_features`, which might add transport features
      * set `VIRTIO_CONFIG_S_FEATURES_OK` to the status register and make sure
      	the device acks it
    * finally, the driver has a chance to probe
      * each driver is different, but some common practices
      * `>find_vqs` to initialize `struct virtqueue`s and requests IRQs
      * some `>get`s to read from the device config space
      * set `VIRTIO_CONFIG_S_DRIVER_OK` to the status register which readies
      	the device.  It is ok to use virtqueues now.

## virtqueue

* A virtqueue is created with `vring_create_virtqueue`
  * `num` is the queue size and is usually between 16 to 256
  * `vring_align` is the size of L1 cacheline
* A vring buffer is allocated
  * it usually takes 1 or 2 pages of DMA-able memory
  * it has a descriptor table
    * `num` of `struct vring_desc` descriptors
  * an available ring
    * 3 of `u16` for bookkeeping
    * `num` of `u16` for descriptor heads
  * an used ring
    * another 3 of `u16` for bookkeeping
    * `num` of `struct vring_used_elem` for descriptor heads
  * when the driver wants to send a buffer to the device
    * it fills in a slot in the descriptor table
    * writes the descriptor index into the available ring
    * notifies the device
  * when the device has finished the buffer
    * it writes the descriptor index into the used ring
    * sends an interrupt
* two callbacks, `notify` and `callback` should be providied, depending on
  whether the communication is one-way or two-way
  * after the driver writes new data, `notify` is called.  This usually
    triggers an MMIO to notify the device
  * after the device sends a virtqueue IRQ, `callback` is called in IRQ
    context.
* vring internals
  * the driver prepares a "buffer"
    * the buffer is usually a struct for the driver
    * the struct is followed by some "out" data that is read-only to the
      device
    * the "out" data is followed by some "in" data that is write-only to the
      device and is for device response
  * when the buffer is ready, it is added to the vring
    * `virtqueue_add_sgs` expects the buffer to be described as an arary of
      `struct scatterlist`
    * each node of each `struct scatterlist` requires a `vring_desc`
      descriptor
      * the descriptor saves the gpa/len of the page
    * the index of the descriptor chain head is written to the avail ring
  * the device is notified with `virtqueue_kick`
  * the device interrupts the guest when the buffer is retired
  * the driver calls `virtqueue_get_buf` to get the buffer back
    * the vring descriptors become free again
  * the driver retires the buffer
    * it can look at the "in" data from the device to know the status
  * virtio 1.1 introduces packed vring

## PCI transport

* a PCI virtio device has an IRQ for config space change 
* it also has per-vq IRQs for the virtqueues
  * or a global virtqueue IRQ for legacy devices

## vhost

* in-kernel virtio device implementation for host kernel
  * it is partial implementation that only processes virtqueue buffers
* to use a vhost driver, the userspace should
  * create a virtio device as it normally would
  * open `/dev/vhost-blah`
  * `VHOST_SET_OWNER`
    * the vhost driver spawns a `vhost-<pid>` thread
  * `VHOST_GET_FEATURES`
    * get the virtio device features the vhost driver supports
  * `VHOST_SET_FEATURES`
    * tell the vhost driver the negotiated features
  * `VHOST_SET_MEM_TABLE`
    * tell the vhost driver the hva-gpa mappings
  * for each virtqueue
    * set up an eventfd pair for sending irqs
      * give the write end to the vhost driver using `VHOST_SET_VRING_CALL`
      * give the read end to KVM
    * set up an eventfd pair for receiving notifications
      * give the write end to KVM
      * give the read end to the vhost driver using `VHOST_SET_VRING_KICK`
    * `VHOST_SET_VRING_x` to describe the vring to the vhost driver
* the vhost driver creates a thread `vhost-<pid>` to drive the virtqueues
  * when the guest driver kicks, the thread is woken up to process the guest
    buffers
  * when it has buffers for the guest, the thread is woken tup to send the
    buffers

## vhost-user

* it is similiar to vhost except the device is implemented by a
  process separated from the hypervisor
* the transfort is a UNIX socket rather than a misc dev `/dev/vhost-blah`
* the protocol mirrors the vhost ioctls
  * the main issue is the hva-gpa mapping
  * the guest physical memory should be backed by a memfd in host, such that
    it can be sent to the other process
* no kernel support needed

## virtio-vhost-user (experimental and appears dead)

* this is a virtio device
* it provides a new transport method, virtio-vhost-user to replace UNIX socket
  in vhost-user
* this allows the device to be implemented by a process in another VM

## VFIO

* a device driver that exposes the device to userspace for direct access
* e.g., vfio-pci is a universal PCI device driver
  * given a PCI device, we need to ask the kernel to unbind the current driver
  * after ubind, we ask the kernel to bind it to vfio-pci driver
* device passthrough
  * VM is in userspace, unprivileged
  * VMM can passthrough the device to VM, and let the VM drives it
* SR-IOV
  * the physical device is referred to as PF, Physical Function
  * its driver can ask it to allocate a virtual device, VF
  * we want vfio-pci rather than the real driver to drive VF
  * this way, we can pass through the VF to the VM
  * VM can drive the VF using the real driver
* how about devices that don't support SR-IOV?
  * the real driver for the physical device can add support for mdev, or
    mediated devices
  * the driver can allocate mdev
  * vfio-mdev can drive the mdev
  * Done.
  * it is the real driver's job to make mdevs appears as independent devices
* it seems people want this now
  * the real driver supports mdev and emulates it as a virtio device
  * the benefit is that the guest only needs virtio drivers

## virtio devices

 - commonly implemented as PCI devices 
 - `virtqueue` is the fundemental building block for bulk data transport
 - there are three common ways to process `virtqueue` operations
   - by the userspace hypervisor/VMM itself
   - by the kernel (vhost) with userspace VMM's help
     - Using the information from KVM, VMM
       - tells vhost the memory mapping of the device
       - registers to vhost a "kick" eventfd, which is for guest->host
       	 notification known as ioeventfd 
       - registers to vhost a "call" eventfd, which is for host->guest
       	 notification known as irqfd
   - by another process (vhost-user)
     - same as vhost, but talks to another process rather than kernel

## virtio drivers

 - because virtio devices are implemented as PCI devices, there is a PCI
   driver, `virtio-pci`, that puts PCI devices onto `virtio` bus
 - virtio drivers are for devices on `virtio` bus

## A legacy PCI virtio device

* Three BARs
  * BAR0 1KB PIO
  * BAR1 1KB MMIO (provides the same funcionality as the PIO)
  * BAR2 512B MSI MMIO
* MMIO registers
  * `VIRTIO_PCI_HOST_FEATURES (0)`
    * get the host features
  * `VIRTIO_PCI_GUEST_FEATURES (4)`
    * set the guest features
  * `VIRTIO_PCI_QUEUE_SEL (14)`
    * select the vq (for use by other vq operations)
  * `VIRTIO_PCI_QUEUE_NUM (12)`
    * get the size of the selected vq
  * `VIRTIO_PCI_QUEUE_PFN (8)`
    * write with >0 guest pfn initializes the vq; the pfn tells the host the
      address of `struct vring`
      * `vring` starts with `QEUEU_NUM` `struct vring_desc`, followed by
      	`struct vring_avail` followed by `struct vring_used`
    * write with 0 shuts down a vq
    * read returns the guest pfn
  * `VIRTIO_PCI_QUEUE_NOTIFY (16)`
    * write indicates that there are new data in the ring buffer
  * `VIRTIO_PCI_STATUS (18)`
  * `VIRTIO_PCI_ISR (19)`
    * to ack an irq
  * `VIRTIO_MSI_CONFIG_VECTOR (20)`
  * `VIRTIO_MSI_QUEUE_VECTOR (22)`

## A modern PCI virtio device

* guest-to-host
  * guest writes to BAR2 @ 12KB
  * vcpu thread triggers userspace exit and ask the device thread to parse the
    vq
* host-to-guest
  * device thread kicks the main thread to triggers `KVM_SIGNAL_MSI`
  * vcpu thread wake up to handle the irq in the guest
