Kernel USB Mass Storage
=======================

## Overview

- `module_usb_stor_driver` defines `usb_storage_driver`
  - `usb_stor_host_template_init` inits `usb_stor_host_template`
  - `usb_storage_usb_ids` matches unusual vendor/product ids and these
    subclasses
    - `USB_SC_RBC`
    - `USB_SC_8020` (CD-ROM)
    - `USB_SC_QIC`
    - `USB_SC_UFI`
    - `USB_SC_8070`
    - `USB_SC_SCSI`
- `storage_probe`
  - `usb_stor_probe1` calls `scsi_host_alloc` to alloc a scsi host
  - `usb_stor_probe2` calls `scsi_add_host` to add a scsi host
    - it also schedules `usb_stor_scan_dwork` to call `scsi_scan_host` to scan
      scsi devices
- `scsi_scan_host` calls `scsi_scan_channel` to scan all targets
  - `scsi_probe_and_add_lun` probes a target and adds a device
    - `scsi_probe_lun` probes the target
    - `scsi_add_lun` adds the device
  - for an optical driver, `sr_probe` binds to the device
    - `register_cdrom` exposes the std cdrom interface to the userspace
      - `sr_dops` provides the ops
    - `sr_bdops` provides the block device ops
      - `sr_block_ioctl` handles ioctls
