Linux scsi
==========

## Specs

- SCSI was initially parallel interface
- modern SCSI development focuses on serial interface
- Serial Attached SCSI
  - SAS-1:  3.0 Gb/s, 2004
  - SAS-2:  6.0 Gb/s, 2009
  - SAS-3: 12.0 Gb/s, 2013
  - SAS-4: 22.5 Gb/s, 2017
  - SAS-5: 45.0 Gb/s, wip
  - SCSI command set
  - host controller usually needs its own driver
- USB Attached SCSI
- SCSI command protocol
  - outside SCSI, ATAPI or USB mass storage also uses SCSI command protocol

## AHCI and SCSI

- SCSI
  - for each `ata_port`, `scsi_host_alloc` is called to allocate a `Scsi_Host`
    - `scsi_host_template` is `AHCI_SHT`, whose queuecommand is
      `ata_scsi_queuecmd`
    - `scsi_add_host_with_dma` calls `scsi_mq_setup_tags` to set up the tag
      set
  - for each enabled `ata_device`, `__scsi_add_device` is called to allocate a
    `scsi_device`
    - `scsi_alloc_sdev` allocates the `scsi_device`
    - `scsi_probe_lun` probes the device
- life of a request
  - `scsi_probe_lun` calls `scsi_execute_req` to execute a SCSI command
  - `__scsi_execute`
    - `blk_get_request` returns a request from the request queue
    - the scsi command is filled in to the request
    - `blk_execute_rq` executes and waits the request
  -  the request is eventually queued by `scsi_queue_rq`
    - when the `scsi_host` was set up, `scsi_mq_setup_tags` set up
      `blk_mq_tag_set` to use `scsi_queue_rq`
    - `scsi_cmnd` is extraced from the request and dispatched by
      `scsi_dispatch_cmd`
    - queuecommand is `ata_scsi_queuecmd`.  `ata_scsi_translate` translates
      `scsi_cmnd` to `ata_queued_cmd`
    - `qc_issue` is `ahci_qc_issue` and issues `ata_queued_cmd` to the HBA
  - when the request is from filesystem, `scsi_cmnd` is uninitialized
    - `scsi_mq_prep_fn` is called to initialize `scsi_cmnd`
    - `scsi_setup_fs_cmnd` calls the driver's `init_command`
      - for sd, it is `sd_init_command`
- sd
  - `scsi_host` or `scsi_device` are devices on the scsi bus in the device
    model
  - sd is a scsi driver in the device model
  - `sd_probe` checks if a `scsi_device` has `TYPE_DISK` and binds to it
  - `alloc_disk` is called to allocate a `gendisk` with name specified by
    `sd_format_disk_name`
    - i.e., sda, sdb, etc.
  - when requests are made against the gendisk, sd is responsible for
    translating the requests into scsi commands

## SCSI Drivers

- an `scsi_driver` drives an `scsi_device`
  - it drives a device, not the controller
- the main `scsi_driver`s are
  - `sd_template`, for `TYPE_DISK` and etc.
  - `sr_template`, for `TYPE_ROM` and `TYPE_WORM`
  - `st_template`, for `TYPE_TAPE`
  - `ses_template`, for `TYPE_ENCLOSURE`
  - `ch_template`, for `TYPE_MEDIUM_CHANGER`
- drivers translate fops (read/write/ioctl) to scsi cmds
  - they also support `SG_IO` ioctl to passes through scsi cmds from userspace
    to devices
    - it only supports a subset of sg v3
    - it is used for
      - device info query
      - smart
      - hdparm
      - etc.
- there is `scsi_bsg_register_queue` that adds a `bsg_device` to all
  `scsi_device`
  - it passes through scsi cmds from userspace to devices
  - it only supports sg v4 ioctl
- there is also `sg_interface` that binds to all `scsi_device`
  - it supports older sg v1, v2, and v3 ioctls
