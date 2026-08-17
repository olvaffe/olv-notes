# smartctl

## SMART

- SATA
  - cmd 0xB0: SMART
    - subcmd 0xD0: Read Data
      - there is a standard 512-byte data, but all attrs are vendor-defined
    - subcmd 0xD5: Read Log
      - there are various logs
    - subcmd 0xDA: Return Status
  - cmd 0xEC: IDENTIFY DEVICE
- SCSI
  - cmd 0x12: INQUIRY
    - standard page
    - vpd (vital product data) pages
  - cmd 0x4D: LOG SENSE
    - various logs
- NVME
  - admin cmd 0x02: Get Log Page
    - log id 0x01: Error
    - log id 0x02: SMART / Health
    - log id 0x06: Self-Test
  - admin cmd 0x06: Identify Controller

## smartctl options

- `-i` queries dev info
  - model, version, capacity, etc.
- `-c` queries dev caps
  - power states, logs, etc.
- `-A` queries attrs
  - temp, tbw, hours, etc.
- `-H` queries health summary
 -  pass/fail, etc.
- `-l <log>` queries logs
  - `error` for errors
  - `selftest` for selftest results
