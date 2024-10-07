Kernel UFS
==========

## Universal Flash Storage Versions

- 1.0, 2011, max 300 MB/s
- 1.1, 2012
- 2.0, 2013, max 1200 MB/s
- 2.1, 2016
- 2.2, 2020
- 3.0, 2018, max 2900 MB/s
- 3.1, 2020
- 4.0, 2022, max 5800 MB/s

## Core

- the core calls `scsi_host_alloc` to add scsi hba
  - the flashes will show up as `/dev/sdX`
