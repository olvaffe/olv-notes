Block Subsystem
===============

## Block Layer

- I/O requests (`struct request`) reach the block layer
- some drivers execute the block requests directly
  - NVM-e
  - virtio-blk (in the guest)
- Some drivers need the requested be translated to another command set first
  - `sd_init_command` of sd

## NVMe

- high-end ssds are connected to PCIe bus,  with non-standard command
  protocols initially
- NVMe is a standardized command protocol
- in the NVM subsystem
  - there can be multiple domains
  - each domain can have multiple controllers
  - each controller can have multiple namespaces
  - each namespace is allocated from the underlying storage media and can be
    formatted into logical blocks
- '/dev/nvme0' is the character device for a controller
- '/dev/nvme0n1' is the block device for a namespace

## AHCI

- Advanced Host Controller Interface
  - a standard interface for SATA controllers
  - the controller is called HBA, host bus adapter
- an HBA can have
  - up to 32 ports, each can connect to a device or a multiplier
  - up to 32 slots, each holds a command
- an `ata_host` represents an AHCI HBA
  - allocated with `ata_host_alloc_pinfo`
  - a `ata_port` is also allocated for each port
- `ahci_host_activate` activates an AHCI HBA
  - `ata_host_register` is called
  - `ata_tport_add` is called for each port to add a "ata%d" device under HBA
  - `ata_tlink_add` is called for each `port->link` to add a "link%d" device
    under port
  - `ata_tdev_add` is called for each `link->device` to add a "dev%d.%d"
    device under link
  - `ata_scsi_add_hosts` is called to allocate a `Scsi_Host` for each port
  - `async_port_probe` is scheduled for each port
    - `ata_port_probe` is called to detect `ata_device` on the link of the
      port
      - this calls `__ata_port_probe` to trigger `ata_std_error_handler`
      - `ata_std_postreset` prints the link status
      - `ata_configure_dev` configures and prints the device status
    - `ata_scsi_scan_host` is called
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

## Filesystem

- `mknod` creates an inode with `def_blk_fops` as the fops
- whenever the inode is opened, `blkdev_open` looks up the corresponding
  `block_device` from another inode in the internal `bdev` fs.
  - `__blkdev_get` finds the `gendisk` associated with the `block_device`,
    optionally allocating the disk on-demand
- the driver (e.g., sd) should do
  - `register_blkdev` adds a `blk_major_name` to `major_names` table
    - for tracking
  - `blk_register_region` adds a probe function to `bdev_map` table
    - the probe function can call `alloc_disk` to allocate a `gendisk`
      on-demand
- `blkdev_write_iter` of `def_blk_fops` calls `__generic_file_write_iter`
  - the page cache (`address_space`) sits in-between the filesystem and the
    block layer
  - `blkdev_writepage` of `def_blk_aops` writes a dirty page to the block
    device
    - it is hard to track down what's going on
    - at the bottom, it is `bio_alloc`, `submit_bio`, and
      `blk_mq_make_request`

## gendisk

- `alloc_disk`
- `disk->queue` `blk_mq_init_queue`
- `disk->disk_name`
- `disk->fops`
- `device_add_disk`

## virtio-blk

- TBD

## Buses

- Serial ATA
  - 3.0: 6 Gbit/s
  - ATA8-ACS command set
  - host controller usually provides a generic AHCI interface
- Serial Attached SCSI
  - SAS-4: 22.5 Gbit/s
  - SCSI command
  - host controller usually needs its own driver
- PCI-e
  - 3.0 x4: 4GB/s
  - NVMe command set
  - host controller usually provides a generic NVMe interface
- Over fabrics
  - iSCSI: SCSI command over TCP/IP
  - the host is a client.  It is called an initiator.
  - the remote has a server.  Each device on the remote is called a target.
