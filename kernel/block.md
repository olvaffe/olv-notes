# Block Subsystem

## Buses

* Serial ATA
  * 3.0: 6 Gbit/s
  * ATA8-ACS command set
  * host controller usually provides a generic AHCI interface
* Serial Attached SCSI
  * SAS-4: 22.5 Gbit/s
  * SCSI command
  * host controller usually needs its own driver
* PCI-e
  * 3.0 x4: 4GB/s
  * NVMe command set
  * host controller usually provides a generic NVMe interface
* Over fabrics
  * iSCSI: SCSI command over TCP/IP
  * the host is a client.  It is called an initiator.
  * the remote has a server.  Each device on the remote is called a target.

## Block Layer

* I/O requests reach the block layer
* some drivers execute the block requests directly
  * NVM-e
  * virtio-blk (in the guest)
* Some drivers need the requested be translated to another command set first
