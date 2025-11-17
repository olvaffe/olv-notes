Linux nvme
==========

## Buses

- NVMe (over PCI-e)
  - 3.0 x4:  32 Gb/s
  - 4.0 x4:  64 Gb/s
  - 5.0 x4: 128 Gb/s
  - NVMe command set
  - host controller usually provides a generic NVMe interface
- NVME-oF (over Fabrics)
  - iSCSI: SCSI command over TCP/IP
  - the host is a client.  It is called an initiator.
  - the remote has a server.  Each device on the remote is called a target.

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
- `/dev/nvme0` is the character device for a controller
- `/dev/nvme0n1` is the block device for a namespace
