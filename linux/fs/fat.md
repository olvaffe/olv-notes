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
