Kernel FAT
==========

## Encodings

- short filenames
  - limited to 8+3 characters
  - encoded in one of the codepages
- long filenames
  - limited to 255 characters
  - encoded in utf-16
- mount options
  - `codepage` is for short filenames
    - default to `CONFIG_FAT_DEFAULT_CODEPAGE` (437)
    - `sbi->nls_disk = load_nls("cp%d")`
    - it converts short filenames between codepage and utf-16, such that short
      filenames can be handled as long filenames
  - `iocharset` is for vfs representation
    - default to `CONFIG_FAT_DEFAULT_IOCHARSET` (`iso8859-1`)
    - `sbi->nls_io = load_nls(sbi->options.iocharset)`
    - it converts filenames between iocharset and utf-16, such that vfs sees
      iocharset instead of utf-16
  - `utf8` hijacks iocharset and utf-16 conversion
    - default to `CONFIG_FAT_DEFAULT_UTF8` (it should be set)
    - it hijacks the conversion such that vfs sees utf-8 instead of iocharset
      - iocharset is still used internally for case-insensitive cmp
- `fat_readdir`
  - `fat_parse_long` reads the utf-16 long filename from the disk
    - `fat16_towchar` is `memcpy` unless endian issue
    - `fat_uni_to_x8` converts utf-16 to the target encoding
      - if `utf8`, it calls `utf16s_to_utf8s` to convert utf-16 to utf-8
      - otherwise, it uses `sbi->nls_io` to convert to `iocharset`
  - `fat_parse_short` reads the short filename from the disk
    - `fat_shortname2uni` uses `sbi->nls_disk` to convert from `codepage` to
      utf-16
    - `fat_uni_to_x8` converts utf-16 to the target encoding

## Physical Format

- <https://en.wikipedia.org/wiki/Design_of_the_FAT_file_system>
- there are 4 regions
  - reserved sectors
    - boot sector
    - fs info sector
    - optional sectors
  - fat region
    - file allocation table #1
    - file allocation table #2 (backup)
  - root directory region (fat12/fat16 only)
  - data region
- boot sector
  - 0x000: jump instruction
  - 0x003: oem name (e.g., `mkfs.fat`)
  - bios parameter block (bpb) below
  - 0x00b: bytes per sector, typically 512
  - 0x00d: sectors per cluster
  - 0x00e: number of reserved sectors, typically 32
  - 0x010: number of file allocation tables, typically 2
  - 0x015: media type, 0xf8 for disk
  - 0x020: total number of sectors
  - 0x024: sectors per file allocation table
  - 0x030: sector of fs info, typically 1
  - 0x032: sector of backup boot sectors, typically 6
  - 0x047: volume label (e.g., `NO NAME`)
  - 0x052: fs type (e.g., `FAT32`)
  - boot code
  - 0x1fe: magic (`0x55 0xaa`)
- fs info sector
  - 0x000: magic (`RRaA`)
  - 0x1e4: magic (`rrAa`)
  - 0x1fc: magic (`0x00 0x00 0x55 0xaa`)
- file allocation table
  - fat consists of an array of entries
  - each entry is 12-/16-/32-bit on FAT12/16/32
  - a file is referred to by a singly-linked list of entries
    - data region is divided into identially-sized clusters
    - each entry points to a cluster in the data region
    - each file occupies one or more clusters
