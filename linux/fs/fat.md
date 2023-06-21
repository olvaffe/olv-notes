Kernel FAT
==========

## Encodings

- short filenames
  - limited to 8+3 characters
  - encoded in one of the codepages
- long filenames
  - limited to 255 characters
  - encoded in utf-16
- configs
  - `CONFIG_FAT_DEFAULT_CODEPAGE` defaults to 437
  - `CONFIG_FAT_DEFAULT_IOCHARSET` defaults to `iso8859-1`
  - `CONFIG_FAT_DEFAULT_UTF8` should be set
- mount options
  - `iocharset` is for long filenames
    - `sbi->nls_io = load_nls(sbi->options.iocharset)`
  - `codepage` is for short filenames
    - `sbi->nls_disk = load_nls("cp%d")`
  - `utf8`
  - `uni_xlate`
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
