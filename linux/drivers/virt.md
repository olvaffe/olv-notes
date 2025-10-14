Linux Virtualization Drivers
============================

## Confidential Computing

- CoCo stands for Confidential Computing
- <https://www.arm.com/architecture/security-features/arm-confidential-compute-architecture>
  - there is the normal world
    - EL0: userspace
    - EL1: kernel (guest)
    - EL2: hypervisor (kvm)
  - trustzone introduces the secure world
    - EL0: trusted app
    - EL1: trusted os
    - EL2: secure partition manager
    - EL3: secure monitor
  - CCA introduces the realms
    - EL0: userspace
    - EL1: kernel (guest)
    - EL2: TF-RMM
- Intel
  - TDX
  - SGX
  - TXT
- AMD
  - PSP
  - SEV
- Why
  - in the normal world, hypervisor can access the memory of normal world VMs
  - by moving VMs to realms, hypervisor can no longer access the memory of VMs
- `CONFIG_ARM_CCA_GUEST`
  - it provides an interface for realm userspace to talk to TF-RMM
- `CONFIG_EFI_SECRET`
  - it provides a fs on EFI secret area, which can only be accessed by the
    realm vm
- `CONFIG_TDX_GUEST_DRIVER`
  - it provides an interface for realm userspace to talk to TDX
- `CONFIG_SEV_GUEST`
  - it provides an interface for realm userspace to talk to SEV
- `CONFIG_ARM_PKVM_GUEST`
  - it provides an interface for realm userspace to talk to pKVM
  - pKVM is a poorman's sw implementation of CoCo
