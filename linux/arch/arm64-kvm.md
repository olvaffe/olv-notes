ARM64 KVM
=========

## KVM

- ELs
  - traditionally,
    - EL0: userspace
    - EL1: kernel
  - armv7a has optional hyp mode
    - EL0: userspace
    - EL1: most of kernel
    - EL2: kernel when running hyp code
      - this is also known as split-mode virtualization or nVHE (non-VHE)
  - armv8.1 has vhe (Virtualization Host Extension)
    - EL0: userspace
    - EL2: kernel
  - inside the vm,
    - EL0: guest userspace
    - EL1: guest kernel
      - it traps to EL2 in the host
  - kernel also has hVHE mode, which uses VHE but splits the kernel like in
    hyp
    - EL0: userspace
    - EL1: most of kernel
    - EL2: kernel when running hyp code
- instructions
  - `svc` traps from EL0 to EL1, used for syscalls
  - `hvc` traps from EL1+ to EL2, used for hypercalls
  - `smc` traps from EL1+ to EL3, used for secure monitor calls
- boot
  - cpu init
    - the boot cpu starts in `primary_entry`, calls `init_kernel_el` and
      `set_cpu_boot_mode_flag`
    - the secondary cpus start in `secondary_holding_pen` (or elsewhere),
      calls `init_kernel_el` and `set_cpu_boot_mode_flag`
  - `init_kernel_el`
    - if cpu is in EL1,
      - inits `sctlr_el1` to `INIT_SCTLR_EL1_MMU_OFF`
      - inits `spsr_el1` to `INIT_PSTATE_EL1`
      - inits `elr_el1` to the return addr
      - inits `w0` to `BOOT_CPU_MODE_EL1`
      - `eret` returns to `elr_el1` at the EL specified by `spsr_el1`, which
        is EL1
    - if cpu is in EL2,
      - inits `elr_el2` to the return addr
      - inits `sctlr_el2` to `INIT_SCTLR_EL2_MMU_OFF`
      - inits `hcr_el2` to `#HCR_E2H`
        - this attemps to enable VHE, which allows the kernel to run at EL2
      - inits `vbar_el2` to `__hyp_stub_vectors`
      - reads `hcr_el2` back to check for VHE support
      - inits `sctlr_el1` to `INIT_SCTLR_EL1_MMU_OFF`
      - `__init_el2_nvhe_prepare_eret` inits `spsr_el2` to `INIT_PSTATE_EL1`
      - inits `w0` to `BOOT_CPU_MODE_EL1`
        - if VHE, `BOOT_CPU_MODE_EL2` too
      - `eret` returns to `elr_el2` at the EL specified by `spsr_el2`, which
        is EL1
  - `set_cpu_boot_mode_flag`
    - `__boot_cpu_mode` is statically initialized to `{EL2, EL1}`
    - when all cpus are in EL2, it is set to `{EL2, EL2}`
    - when all cpus are in EL1, it is set to `{EL1, EL1}`
    - when mixed, it is set to `{EL1, EL2}` which is invalid
  - `finalise_el2`
    - cpu is in EL1
    - `hvc` traps to `elx_sync` in EL2 and calls `__finalise_el2`
    - `eret` stays at EL2 if VHE is supported
- `is_hyp_mode_available` returns true when `__boot_cpu_mode` is EL2
  - both VHE and nVHE boot in EL2
  - KVM is disabled when this returns false
- `is_kernel_in_hyp_mode` returns true if the cpu is currently in EL2
  - this is true when VHE
  - false when nVHE or hVHE (split virt), because most of the kernel runs in
    EL1
- `module_init(kvm_arm_init)`
  - if doing split virt, most of the kernel runs in EL1 and `init_hyp_mode`
    preps EL2 to run hyp code
- `KVM_CREATE_VM` ioctl
  - `kvm_arch_init_vm`
    - `kvm->arch` has arch-specific type `kvm_arch`
    - `kvm_share_hyp` shares the `kvm` struct with EL2 address space
      - nop if already in EL2
    - `kvm_init_stage2_mmu` inits `kvm->arch.mmu`
      - `kvm_pgtable_stage2_init`
  - `kvm_arch_hardware_enable`
    - if split virt,
      - `hyp_install_host_vector` installs `__kvm_hyp_init` vector for EL2
      - `arm_smccc_1_1_hvc(...)` traps to `__do_hyp_init` in EL2 and calls
        `___kvm_hyp_init`
        - this installs `__kvm_hyp_host_vector` vector for EL2
      - `cpu_set_hyp_vector` installs `__kvm_hyp_vector` vector for EL2
  - `kvm_arch_post_init_vm`
- `KVM_CREATE_VCPU` ioctl
  - `kvm_arch_vcpu_precreate`
  - `kvm_arch_vcpu_create`
  - `kvm_arch_vcpu_postcreate`
- `KVM_ARM_VCPU_INIT` ioctl
- `KVM_SET_USER_MEMORY_REGION2` ioctl
  - `kvm_arch_prepare_memory_region`
  - `kvm_arch_commit_memory_region`
  - when guest accesses an addr whose stage-2 mapping hasn't been setup,
    `kvm_handle_guest_abort` is called
    - `kvm_pgtable_stage2_map` sets up stage-2 mapping
- `KVM_RUN` ioctl
  - `kvm_arch_vcpu_ioctl_run` 
    - `kvm_arm_vcpu_enter_exit` calls `__kvm_vcpu_run`
      - two variants depending on whether VHE or nVHE/hVHE

## pKVM

- <https://source.android.com/docs/core/virtualization/architecture>
- KVM VHE mode (since ARMv8.1)
  - EL0: guest usrespace
  - EL1: guest kernel
  - EL2: hypervisor (host kernel)
- KVM nVHE mode (before ARMv8.1)
  - EL0: guest userspace
  - EL1: guest kernel and host kernel w/o KVM
  - EL2: host kernel with only KVM
- pKVM
  - EL0: guest userspace
  - EL1: guest kernel and host kernel w/o KVM and stage-2 mmu
  - EL2: host kernel with only KVM and stage-2 mmu
- <https://android.googlesource.com/kernel/common/+/68468ba8cd5192d948eaac7759c74e940497e995> and
  <https://android.googlesource.com/kernel/common/+/812696a58736b969e7137b7fa07c78f5f2292e36>
  - host kernel boots with `kvm-arm.mode=protected` in EL2
  - it splits out pKVM code remaining in EL2 and deprivileges the rest to EL1
  - host kernel (in EL1) and its userspace is known as host VM
  - `KVM_CREATE_VM` creates a normal guest VM
  - `KVM_CREATE_VM(KVM_VM_TYPE_ARM_PROTECTED)` creates a protected guest VM
