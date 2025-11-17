Kernel UFS
==========

## Universal Flash Storage Versions

- UFS, Universal Flash Storage
  - there are both UFS memory card and eUFS
- 1.0, 2011, max 300 MB/s
- 1.1, 2012
- 2.0, 2013, max 1200 MB/s
- 2.1, 2016
- 2.2, 2020
- 3.0, 2018, max 2900 MB/s
- 3.1, 2020
- 4.0, 2022, max 5800 MB/s

## Host Controller Driver

- `ufshcd_alloc_host` allocs a `ufs_hba`
  - `scsi_host_alloc` allocs a `Scsi_Host` with `ufs_hba` as priv data and
    `ufshcd_driver_template` as template
- `ufshcd_init` inits `ufs_hba`
  - `ufshcd_hba_init` inits hba
  - `ufshcd_hba_capabilities` reads hba caps
  - `ufshcd_device_reset` resets the attached dev
  - `ufshcd_hba_enable` enables hba
  - `ufshcd_link_startup` starts up the link
  - `ufshcd_verify_dev_init` pings the attached dev
  - `ufshcd_complete_dev_init` waits for dev ready
  - `ufshcd_device_params_init` inits dev params
  - `ufshcd_post_device_init` post-inits dev
  - `ufshcd_add_scsi_host`
    - `scsi_add_host` adds `Scsi_Host` to scsi subsys
  - `async_schedule(ufshcd_async_scan, hba)` schedules `ufshcd_async_scan`
- `ufshcd_async_scan`
  - `ufshcd_probe_hba` probes attached devs (really?)
  - `ufshcd_add_lus`
    - it calls `__scsi_add_device` to add 3 `scsi_device`
      - `hba->ufs_device_wlun` at `UFS_UPIU_UFS_DEVICE_WLUN`
      - `sdev_rpmb` at `UFS_UPIU_RPMB_WLUN`
      - `sdev_boot` at `UFS_UPIU_BOOT_WLUN`
    - `__scsi_add_device` also
      - `scsi_alloc_target` allocs `scsi_target`
      - `scsi_probe_and_add_lun` probes and creates `scsi_device`
    - `scsi_scan_host` is unnecessary?
- modules such as `sd` provides an `scsi_driver` to probe `scsi_device`
  - the flashes will show up as `/dev/sdX`
