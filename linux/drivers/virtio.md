virtio
======

## Spec

- <https://github.com/oasis-tcs/virtio-spec>
- <https://docs.oasis-open.org/virtio/virtio/v1.3/virtio-v1.3.html>
- 2 Basic Facilities of a Virtio Device
  - 2.1 Device Status Field
    - mainly set by guest to report initialization stages
      - `ACKNOWLEDGE` when device is discovered
      - `DRIVER` when driver is bound
      - `FEATURES_OK` when feature negotication has completed
      - `DRIVER_OK` when driver is fully set up
      - `FAILED` when driver can't support the device
    - device can set `DEVICE_NEEDS_RESET`
  - 2.2 Feature Bits
    - bit  0..23: specific to device type
    - bit 24..37: reserved for extensions to virtqueue and feature negotiation
      - `VIRTIO_F_VERSION_1` is set for virtio 1.0 compliant devices
    - bit 38..xx: reserved for future extensions
  - 2.3 Notifications
    - device to driver
      - configuration change notification
      - used buffer notification
      - usually implemented as interrupts
    - driver to device
      - available buffer notification
      - usually implemented as mmio
  - 2.4 Device Configuration Space
    - generally used for rarely changing or intialization time parameters
    - read generation before and after for atomicy
    - some regions are optional and are present only when the corresponding
      feature bits are set by device
      - no negotiation required
  - 2.5 Virtqueues
    - mechanism for bulk data transport
    - a device can have zero or more virtqueues
    - driver makes requests by adding available buffers to the queue and
      optionally triggers available buffer notifications
    - device executes requests.  When complete, device adds used buffers back
      to the queue and optionally triggers used buffer notifications
      - requests might be executed and/or completed out-of-order, depending on
      	the device
    - each virtqueue consists of up to 3 parts
      - descriptor area, used for describing buffers
      - driver area, extra data supplied by driver to device
      - device area, extra data supplied by device to driver
  - 2.6 Split Virtqueues
    - each split virtqueue consists of
      - descriptor table
        - `(16*QueueSize)` bytes in size
        - alignment 16
      - available ring
        - `(6+2*QueueSize)` bytes in size
        - alignment 2
      - used ring
        - `(6+8*QueueSize)` bytes in size
        - alignment 4
      - QueueSize is 16-bit and is power-of-2
        - usually read from configuration space
      - because ring size is the same as the descriptor table size, the rings
      	are not full whenever there are still free descriptors
      - each part is physically contiguous in guest and logically contiguous
      	in host
        - in linux, `vring_alloc_queue` uses `dma_alloc_coherent`
    - a descriptor in the descriptor table describes a buffer and consists of
      - `le64 addr` is the guest physical address of the buffer
      - `le32 len` is the size of the buffer
      - `le16 flags` can be
        - `VIRTQ_DESC_F_NEXT` indicates that the buffer continues as described
          by the next descriptor
	  - used when the buffer is larger than a page and not physically
	    contiguous
        - `VIRTQ_DESC_F_WRITE` indicates the buffer is write-only by device
          (otherwise, read-only by device)
        - `VIRTQ_DESC_F_INDIRECT` indicates the buffer contains a list of
          buffer descriptors
          - thus the real buffer can be huge and discontiguous while being
            described by one descriptor in the descriptor table
      - `le16 next` is the index of the next descriptor
      - if `VIRTIO_F_IN_ORDER` is negotiated, driver must use descriptors in
      	order starting from 0
    - the available ring is written by driver and read by device, and consists
      of (see `struct virtq_avail` for the real layout)
      - `le16 idx` is incremented when a buffer is added
        - ring head
        - `ring[idx++ % QueueSize] = buffer_desc_idx`
      - `le16 ring[QueueSize]` where each entry is an index into the
      	descriptor table
      - if `VIRTIO_F_EVENT_IDX` is negotiated,
        - `le16 flags` must be 0
        - `le16 used_event` suppresses used buffer notification until the
          device writes `ring[used_event]` of the used ring
          - equivalently, `idx` of the used ring equals to `used_event+1`
      - if `VIRTIO_F_EVENT_IDX` is not negotiated,
        - `le16 flags`
          - `VRING_AVAIL_F_NO_INTERRUPT` suppresses used buffer notification
        - `le16 used_event` is ignored
    - the used ring is written by device and read by driver, and consists of
      (see `struct virtq_used` for the real layout)
      - `le16 idx`
        - ring head
        - `ring[idx++ % QueueSize] = {id, len}`
      - `le64 ring[QueueSize]` where each entry is a {id, len} pair
        - `id` is the index of the buffer descriptor
        - `len` is the total bytes written by device
      - if `VIRTIO_F_EVENT_IDX` is negotiated,
        - `le16 flags` must be 0
        - `le16 avail_event` suppresses available buffer notification until
          the driver writes `ring[avail_event]` of the available ring
      - if `VIRTIO_F_EVENT_IDX` is not negotiated,
        - `le16 flags`
          - `VIRTQ_USED_F_NO_NOTIFY` suppresses available buffer notification
        - `le16 avail_event` is ignored
    - 2.6.13 Supplying Buffers to The Device
      - the driver places the buffer into free descriptor(s), chaining as
      	necessary
      - the driver places the index of the head of the descriptor chain into
      	the next ring entry of the available ring
      - repeat the first two steps to batch more buffers
      - The driver performs a suitable memory barrier to ensure the device
      	sees the updated descriptor table and available ring before the next
      	step.
      	- a write barrier
      - the available `idx` is increased by the number of descriptor chain
      	heads added to the available ring.
      - The driver performs a suitable memory barrier to ensure that it
      	updates the idx field before checking for notification suppression.
      	- hmm, according to 2.6.13.4.1, this is to see the device's write to
      	  `flags` or `avail_event`
      	- an acquire barrier?
      - The driver sends an available buffer notification to the device if
      	such notifications are not suppressed.
  - 2.7 Packed Virtqueues
    - negotiated by `VIRTIO_F_RING_PACKED`
    - each packed virtqueue consists of
      - Descriptor Ring of a ring of buffer descriptors
        - alignment 4
        - size `(16*QueueSize)`
      - Driver Event Suppression
        - a mechanism to reduce used buffer notifications
        - read-only by device
        - alignment 4
        - size 4
      - Device Event Suppression
        - a mechanism to reduce available buffer notifications
        - write-only by device
        - alignment 4
        - size 4
    - each of the driver and the device maintains a 1-bit ring wrap counter
      - the counters are initialized to 1
      - the one in the driver is called Driver Ring Wrap Counter
	- it is flipped whenever the driver marks the last descriptor in the
	  ring available and wraps to the first descriptor
      - the one in the device is called Device Ring Wrap Counter
        - it is flipped whenever the device marks the last descriptor in the
          ring used and wraps to the first descriptor
      - to mark a descriptor available, the driver should
        - if the driver counter is 1, set `VIRTQ_DESC_F_AVAIL` and unset
          `VIRTQ_DESC_F_USED`
        - if the driver counter is 1, unset `VIRTQ_DESC_F_AVAIL` and set
          `VIRTQ_DESC_F_USED`
      - to mark a descriptor used, the device should
	- if the device counter is 1, set `VIRTQ_DESC_F_AVAIL` and
	  `VIRTQ_DESC_F_USED`
	- if the device counter is 0, unset `VIRTQ_DESC_F_AVAIL` and
	  `VIRTQ_DESC_F_USED`
      - to summarize,
        - when `VIRTQ_DESC_F_AVAIL` and `VIRTQ_DESC_F_USED` differ, the
          descriptor is available
        - when `VIRTQ_DESC_F_AVAIL` and `VIRTQ_DESC_F_USED` agree, the
          descriptor is used (or free)
        - when the driver counter and the device counter agree, 
    - a buffer consists of zero or more read-only physically contiguous
      elements followed by zero or more write-only physically contiguous
      - there is at least one element
      - each element is described by a buffer desciptor
      - same as splitted virtqueue
      - it differs from splitted virtqueue in that
        - instead of `VIRTQ_DESC_F_NEXT`, `id==0` indicates that there are
          more elements described by the following descriptors
	- instead of incrementing `idx` (ring head), `VIRTQ_DESC_F_AVAIL` and
	  `VIRTQ_DESC_F_USED` together decide whether a descriptor is
	  available or used (or free)
    - each buffer descriptor in the descriptor ring consits of
      - `le64 addr` is the physical address of the buffer region
	- a buffer descriptor describes a physically contiguous region that is
	  either read-only or write-only
          - just like in splitted virtqueues
        - a buffer might require mulitple descriptors to describe
          - when it is not physically contigous, or
          - when it has both read-only regions and write-only regions
      - `le32 len` is the size of the buffer region
      - `le16 id` is opaque
        - on linux, it is just the index of the descriptor in the ring
      - `le16 flags`
        - `VIRTQ_DESC_F_WRITE` if device write-only; otherwise, device
          read-only
        - `VIRTQ_DESC_F_AVAIL`
        - `VIRTQ_DESC_F_USED`
    - the event suppression struct consists of
      - `le16 desc`
      - `le16 flags`
- `virtio_config_ops` is the kernel abstraction of operations against a virtio
  device
  - `get_status` and `set_status` manipulate the status field
  - `get_features` and `finalize_features` negotiate feature bits
  - `get` and `set` manipulate the configuration space
  - `find_vqs` and `del_vqs` set up virtqueues
  - `get_shm_region`

## Core

- the core registers a `virtio` bus
  - the bus matches devices with drivers
  - each virtio driver calls `register_virtio_driver` to register itself to
    the bus
  - the virtio-pci driver discovers PCI virtio devices, allocates a
    `virtio_pci_device` which is a subclass of `virtio_device`, and
    `register_virtio_device` the virtio device to the bus
- the core registers a virtio-pci driver for PCI devices
  - it matches all `PCI_VENDOR_ID_REDHAT_QUMRANET` PCI devices
  - for each matched `pci_dev`, it allocates a `virtio_pci_device` which is a
    child of the `pci_dev`
    - it calls `pci_enable_device` to enable the PCI device
    - it calls `virtio_pci_modern_probe` to initialize the virtio device
      - or falls back to `virtio_pci_legacy_probe`
      - maps `common`, `device`, and `notify_base` regions for MMIO
      - initializes `virtio_config_ops` to `virtio_pci_config_ops`, for use by
      	the virtio driver
      - initializes `setup_vq`, `del_vq`, and `config_vector` callbacks to
      	abstract the differences between legacy/modern PCI virtio devices
    - it calls `register_virtio_device` to register the device to the virtio
      bus
- each virtio device has a `virtio_config_ops` to config the device
  - in `register_virtio_device`
    - the device is always `->reset` first when registered
    - then `VIRTIO_CONFIG_S_ACKNOWLEDGE` is set to the status register
  - in `virtio_dev_probe`
    - set `VIRTIO_CONFIG_S_DRIVER` to the status register
    - decide features
      - get device features from the device feature register
      - get driver features from the driver feature table
      - the usable features are those set in both masks
      - `>finalize_features`, which might add transport features
      - set `VIRTIO_CONFIG_S_FEATURES_OK` to the status register and make sure
      	the device acks it
    - finally, the driver has a chance to probe
      - each driver is different, but some common practices
      - `>find_vqs` to initialize `struct virtqueue`s and requests IRQs
      - some `>get`s to read from the device config space
      - set `VIRTIO_CONFIG_S_DRIVER_OK` to the status register which readies
      	the device.  It is ok to use virtqueues now.

## virtqueue

- A virtqueue is created with `vring_create_virtqueue`
  - `num` is the queue size and is usually between 16 to 256
  - `vring_align` is the size of L1 cacheline
- A vring buffer is allocated
  - it usually takes 1 or 2 pages of DMA-able memory
  - it has a descriptor table
    - `num` of `struct vring_desc` descriptors
  - an available ring
    - 3 of `u16` for bookkeeping
    - `num` of `u16` for descriptor heads
  - an used ring
    - another 3 of `u16` for bookkeeping
    - `num` of `struct vring_used_elem` for descriptor heads
  - when the driver wants to send a buffer to the device
    - it fills in a slot in the descriptor table
    - writes the descriptor index into the available ring
    - notifies the device
  - when the device has finished the buffer
    - it writes the descriptor index into the used ring
    - sends an interrupt
- two callbacks, `notify` and `callback` should be providied, depending on
  whether the communication is one-way or two-way
  - after the driver writes new data, `notify` is called.  This usually
    triggers an MMIO to notify the device
  - after the device sends a virtqueue IRQ, `callback` is called in IRQ
    context.
- vring internals
  - the driver prepares a "buffer"
    - the buffer is usually a struct for the driver
    - the struct can point to some "out" data that is read-only to the device
    - the struct can point to some "in" data that is write-only to the device
      and is used for device response
  - the buffer is added to the vring
    - the "out" and "in" data must be described as an array of `struct
      scatterlist *`
    - `virtqueue_add_sgs` adds the data to the vring
      - for "out" only vq, `virtqueue_add_outbuf` can be used
      - for "in" only vq, `virtqueue_add_inbuf` can be used
      - a pointer to the driver "buffer" is also given as a cookie and is
      	returned in `virtqueue_get_buf` when the buffer is retired.
    - each node of each `struct scatterlist` requires a `vring_desc`
      descriptor
      - the descriptor saves the gpa/len of the page
    - the index of the descriptor chain head is written to the avail ring
  - the device is notified with `virtqueue_kick`
  - the device interrupts the guest when the buffer is retired
    - buffers might not be executed in-order nor retired in-order
  - the driver calls `virtqueue_get_buf` to get the buffer back
    - the vring descriptors become free again
  - the driver retires the buffer
    - it can look at the "in" data from the device to know the status
  - virtio 1.1 introduces packed vring

## vring

- a PCI device uses its configuration space to report the maximum queue size
  supported
- a `struct vring` is created
  - `desc` points to an array of `struct vring_desc`
  - `avail` points to the available ring, `struct vring_avail`
  - `used` points to the used ring, `struct vring_used`


## PCI transport

- a PCI virtio device has an IRQ for config space change 
- it also has per-vq IRQs for the virtqueues
  - or a global virtqueue IRQ for legacy devices

## virtio devices

 - commonly implemented as PCI devices 
 - `virtqueue` is the fundemental building block for bulk data transport
 - there are three common ways to process `virtqueue` operations
   - by the userspace hypervisor/VMM itself
   - by the kernel (vhost) with userspace VMM's help
     - Using the information from KVM, VMM
       - tells vhost the memory mapping of the device
       - registers to vhost a "kick" eventfd, which is for `guest->host`
       	 notification known as ioeventfd 
       - registers to vhost a "call" eventfd, which is for `host->guest`
       	 notification known as irqfd
   - by another process (vhost-user)
     - same as vhost, but talks to another process rather than kernel

## virtio drivers

 - because virtio devices are implemented as PCI devices, there is a PCI
   driver, `virtio-pci`, that puts PCI devices onto `virtio` bus
 - virtio drivers are for devices on `virtio` bus

## A legacy PCI virtio device

- Three BARs
  - BAR0 1KB PIO
  - BAR1 1KB MMIO (provides the same funcionality as the PIO)
  - BAR2 512B MSI MMIO
- MMIO registers
  - `VIRTIO_PCI_HOST_FEATURES (0)`
    - get the host features
  - `VIRTIO_PCI_GUEST_FEATURES (4)`
    - set the guest features
  - `VIRTIO_PCI_QUEUE_SEL (14)`
    - select the vq (for use by other vq operations)
  - `VIRTIO_PCI_QUEUE_NUM (12)`
    - get the size of the selected vq
  - `VIRTIO_PCI_QUEUE_PFN (8)`
    - write with >0 guest pfn initializes the vq; the pfn tells the host the
      address of `struct vring`
      - `vring` starts with `QEUEU_NUM` `struct vring_desc`, followed by
      	`struct vring_avail` followed by `struct vring_used`
    - write with 0 shuts down a vq
    - read returns the guest pfn
  - `VIRTIO_PCI_QUEUE_NOTIFY (16)`
    - write indicates that there are new data in the ring buffer
  - `VIRTIO_PCI_STATUS (18)`
  - `VIRTIO_PCI_ISR (19)`
    - to ack an irq
  - `VIRTIO_MSI_CONFIG_VECTOR (20)`
  - `VIRTIO_MSI_QUEUE_VECTOR (22)`

## A modern PCI virtio device

- guest-to-host
  - guest writes to BAR2 @ 12KB
  - vcpu thread triggers userspace exit and ask the device thread to parse the
    vq
- host-to-guest
  - device thread kicks the main thread to triggers `KVM_SIGNAL_MSI`
  - vcpu thread wake up to handle the irq in the guest
