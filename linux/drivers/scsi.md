Linux scsi
==========

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
