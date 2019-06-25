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

## KVM APIs and boot

* open `/dev/kvm`
* `KVM_CREATE_VM` to create a new VM with no CPU nor memory
  * returns a fd to manage the VM
* `KVM_CHECK_EXTENSION` to check for extensions
  * these are required
    * `KVM_CAP_COALESCED_MMIO`
    * `KVM_CAP_SET_TSS_ADDR`
    * `KVM_CAP_PIT2`
    * `KVM_CAP_USER_MEMORY`
    * `KVM_CAP_IRQ_ROUTING`
    * `KVM_CAP_IRQCHIP`
    * `KVM_CAP_HLT`
    * `KVM_CAP_IRQ_INJECT_STATUS`
    * `KVM_CAP_EXT_CPUID`
* `KVM_SET_TSS_ADDR`
* `KVM_CREATE_PIT2`
* mmap with the desired size of VM physical ram
* `KVM_CREATE_IRQCHIP`
* `KVM_SET_USER_MEMORY_REGION`
  * kernel limits
    * address space count: 2 on x86, 1 otherwise
    * slot count: 32 on ARM, 512 on ARM64, 509 on x86
* load kernel and initrd into the guest RAM
  * read the boot parameters from the bzImage
  * load the real-mode boot sector from the bzImage to
    `BOOT_LOADER_SELECTOR+BOOT_LOADER_IP` of the guest RAM
    * `guest_flat_to_host` translates gpa (guest physical address) to hva
      (host virtual address)
  * load the entire bzImage to gpa `BZ_KERNEL_START`
  * copy the user-specified cmdline to gpa `BOOT_CMDLINE_OFFSET`
  * copy the entire initrd to gpa below `initrd_addr_max`, specified by boot
    parameters
  * patch the real-mode boot sector with the initrd and cmdline gpas
* load BIOS (and/or SMP, ACPI) into the guest RAM
  * load the entire bios to gpa `MB_BIOS_BEGIN`
  * set up E820 memory map at gpa `E820_MAP_START`
  * patch vesa oem sring and modes into gpa `VGA_ROM_OEM_STRING` and `VGA_ROM_MODES`
  * set up real-mode irq handlers
  * without ACPI support, set up mptable as defined by MultiProcessor
    Specification (MPS)
    * this step is deferred until vCPUs and devices are added
* add vCPUs; for each vCPU,
  * check `KVM_CAP_MAX_VCPUS` and `KVM_CAP_NR_VCPUS` for optimal CPU count
  * `KVM_CREATE_VCPU` to create vCPUs
    * returns an fd to control the vCPU
  * `KVM_GET_VCPU_MMAP_SIZE` and mmap the vcpu fd.  The returned pointer can
    be casted to `struct kvm_run`, a struct used to for `KVM_RUN` ioctl
    in/out.  There are more data follow the struct.
  * check `KVM_CAP_COALESCED_MMIO` and get the offset of the coalesced mmio
    ring buffer in the mmaped region
    * MMIO writes by guest will be buffered in the ring buffer
  * `KVM_GET_LAPIC` and `KVM_SET_LAPIC` to configure LAPIC for the vCPU
    * for LINT0 (local interrupt vector 0), deliver as external interrupts
    * for LINT1, deliver as NMI, non-maskable interrupts
* `KVM_SET_GSI_ROUTING`
  * GSI stands for global system interrupt, or simply the IRQ number
    * it is further mapped to an index to the CPU interrupt vector
  * x86 has two 8259A PIC, one master and one slave; each has 8 pins
    * IRQ 0-7 are handled by the master PIC
    * IRQ 8-15 are handled by the slave PIC; the output goes to pin 2 of the
      master PIC
  * the second generation PIC is IO APIC; more pins and memory-mappable
  * the third generation PCI is MSI.  This is not a real controller.
    Interrupts are implemented as writes to specific addresses
* for each CPU, start a CPU thread
  * reset the CPU
    * `KVM_GET_SUPPORTED_CPUID` and `KVM_SET_CPUID2`
    * `KVM_GET_SREGS` and `KVM_SET_SREGS`
    * `KVM_SET_REGS`
      * specificially, eip tells CPU where to begin and esp tells CPU where the
        stack is
    * `KVM_SET_FPU`
    * `KVM_SET_MSRS`
  * enter a loop to call `KVM_RUN` repeateadly.  When `KVM_RUN` returns, it is
    a VM exit and there is a reason that needs to be simulated
    * `KVM_EXIT_IO` if guest reads/writes an IO port
    * `KVM_EXIT_MMIO` if guest reads/writes an MMIO address
    * `KVM_EXIT_SYSTEM_EVENT` if guest triggers reboot or shutdown

## KVM Devices

* PIO and MMIO
  * when the guest reads/writes an I/O port, it exits VM.  When the kernel
    cannot emulate it, the kernel exits `KVM_RUN`  with `KVM_EXIT_IO`
    * `KVM_EXIT_IO_IN`: guest executes IN instruction
    * `KVM_EXIT_IO_OUT`: guest executes OUT instruction
  * when the guest reads/writes to gpa that is not backd by guest physical
    memory, it is assumed to be backed by MMIO (usually device registers).  If
    the kernel cannot emulate it, it exits `KVM_RUN` with `KVM_EXIT_MMIO`.
  * `KVM_IOEVENTFD` is an alternative to `KVM_EXIT_IO` or `KVM_EXIT_MMIO`.
    When the guest writes a pio or mmio address that is in the registered
    region, the corresponding eventfd is set.  This is useful when the
    PIO/MMIO only needs to trigger a notification and can be handled
    asynchronously.
* IRQ
  * `KVM_IRQ_LINE` sets IRQ line high or low explicitly
  * `KVM_IRQFD` injects IRQ to he guest whenever the eventfd is set by the
    host kernel or the userspace.
  * MSI
    * Traditionally, a device has an interrupt line (pin) which is asserted
      when the device wants to signal an interrupt.  This is out-of-band
      because it requires an extra pin rather than the main data path.
    * With MSI, the device writes an interrupt descriptor to a special MMIO
      address to signal an interrupt.  The signaling is in-band with the
      data path.
    * The device has a BAR for MSI MMIO.  This allows the driver to program
      the special MMIO address for the device to signal interrupts.
    * `KVM_SIGNAL_MSI` triggers MSI explicitly
* PCI
  * ioports at `PCI_CONFIG_DATA (0xcfc)` and `PCI_CONFIG_ADDRESS (0xcf8)`
    * address is 32-bit composed of bus/dev/func/reg numbers together with an
      offset into the 256-bytes pci configuration space
    * data is 256-bytes pci deivce header, which are accessed with address
  * mmio with 24-bit CONFIG region that can replace ioports
    * offset into the region has 24 bits, which is used to reconstruct the
      bus/dev/func/reg numbers and others to identify a device function
  * guest uses the configuration space to find out...
    * BAR sizes needed by the PCI device function
    * tell the PCI device function where each BAR is mapped to in the gpa
  * PCI configuration space has interrupt pin and interrupt line fields
    * pin is INTA#...INTD#.  Just use INTA#.
    * line is the IRQ number 0..15
  * Example: PCI device with guest mappable device memory
    * there is a BAR of the size of the device memory for mapping in guest
    * the storage of the device memory is specified with
      `KVM_SET_USER_MEMORY_REGION`, like physical memories
* VFIO can assign host physical PCI devices to guests (PCI passthrough)
* virtio-dev

## IRQ

* i8042
  * to simulate a keyboard press, set the irq line high
  * the guest will be interrupted.  It will read the status register, which
    should indicate that there is keyboard data
  * the guest will read the data register, which should set the irq line low

## `KVM_SET_USER_MEMORY_REGION`

* the ioctl turns a host virtual memory region into a guest physical memory region
  * it creates `kvm_memory_slot`
  * it specifies the `gpa->hva` mapping
* when the guest acceeces a gpa in the region for the first time
  * it triggers an EPT violation
    * `handle_ept_violation` -> `kvm_mmu_page_fault` -> `tdp_page_fault`
  * indirectly in `__gfn_to_pfn_memslot`,
    * it find the `kvm_memory_slot` from the gpa
    * it maps gpa to hva using the slot
    * it pages in the page pointed to by hva and return the hpa
    * for MMIO that has no page, it walks the paging table to find the hva
  * with gpa and hva, the shadow page table entry can be updated
* when the host updates the vma of the host memory (e.g, swap in/out, move)
  * the registered notifier callback calls `kvm_set_pte_rmapp`
  * it updates the spte to point to the new hpa
