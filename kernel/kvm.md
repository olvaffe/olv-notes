# KVM

## Virtualization

* software-based full virtualization
  * hypervisor/VMM is a regular userspace process
  * guest privileged instructions trap (SIGILL) into VMM
    * VMM simulates the appropriate behaviors
    * including guest syscalls into guest kernel
  * guest page table updates trap (SIGSEGV) into VMM
    * guest is given a shadow page table that is marked readonly
    * guest updates trap into VMM
    * VMM updates the real page table to map guest virtual address to host
      physical address
  * guest device io trap (SIGSEGV) into VMM as well
* hardware-assisted full virtualization
  * AMD-V or Intel VT-x
    * first generation adds instructions to enter/exit VM execution mode
      * when in VM execution mode, guest can execute privileged instructions
    * second generation adds nested paging, aka Second Level Address Translation
      (SLAT)
      * a second-level page table to transalte guest virtual address to guest
        physical address, which is treated as host virtual address and goes
        through the first-leve page table to map to the host physical address
  * interrupt virtualization (AMD AVIC and Intel APICv)
  * GPU virualization (Intel GVT-x)
  * IOMMU virualization (AMD-Vi and Intel VT-d)
  * network virualization (Intel VT-c)
  * PCI-SIG Single Root I/O Virtualization (SR-IOV)
* Paravirtualization
  * the guest must be aware and support the virtualization technology used by VMM
* OS-level virtualization: containers
  * based on cgroups and namespaces
    * cgroups provide resource metering and limiting
      * CPU, memory, block io, network, /dev
    * namespaces provide subsystem isolations
      * pid, net, mnt, uts, ipc, user
  * feels like a VM, but not a VM
    * e.g., use host kernel

## KVM Overview

* KVM provides hardware-assisted full virtualization
  * in-kernel, aka native/type-1 virtualization rather than hosted/type-2 virtualization
* KVM has paravirtualization support for devices through virtio
  * there must be a guest driver for each virtio device
* There must be a userspace component (so not exactly a type-1 hypervisor)
  * qemu
  * kvmtool, https://git.kernel.org/pub/scm/linux/kernel/git/will/kvmtool.git/
  * crosvm
  * novm
* TODO: read kvmtool to learn KVM
