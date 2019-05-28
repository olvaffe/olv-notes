# virtio

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

## A PCI virtio device

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
