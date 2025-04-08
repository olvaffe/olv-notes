DRM AMDKFD
==========

## amdkfd

- in module init, `amdgpu_init` calls `amdgpu_amdkfd_init`
  - this initializes the kfd driver
  - `kfd_chardev_init` creates `/dev/kfd`
  - `kfd_procfs_init`
  - `kfd_debugfs_init` creates `/sys/kernel/debug/kfd`
- in dev init,
  - `amdgpu_device_ip_early_init` calls `amdgpu_amdkfd_device_probe` to
    initialize `adev->kfd.dev`
  - `amdgpu_device_ip_init` calls `amdgpu_amdkfd_device_init`
- after dev init, `amdgpu_pci_probe` calls `amdgpu_amdkfd_drm_client_create`
  - `drm_client_init` opens the drm device (as an in-kernel user)
- `kfd_open` is called when `/dev/kfd` is opened
  - `filep->private_data` is set to `kfd_process`
- `amdkfd_ioctls` is the ioctl table for `/dev/kfd`
  - `kfd_ioctl_create_queue` creates a queue
    - `pqm_create_queue` calls `dev->dqm->ops.register_process`
    - `register_process` calls `kfd_inc_compute_active`
    - `kfd_inc_compute_active` calls `amdgpu_amdkfd_set_compute_idle`
    - `amdgpu_amdkfd_set_compute_idle` calls `amdgpu_dpm_switch_power_profile`
    - `amdgpu_dpm_switch_power_profile` calls `pp_funcs->switch_power_profile`
    - `smu_switch_power_profile` calls
      `smu->ppt_funcs->set_power_profile_mode`
