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
 - 
