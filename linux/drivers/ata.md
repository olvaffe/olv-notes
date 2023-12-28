Linux ata
=========

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
