Zstandard
=========

## zstd command line

- usage
  - `zstd <files>...` compresses specified files individually
  - `zstd` or `zstd -` compresses stdin to stdout
    - stdin/stdout must not be tty unless `-f` or `-c`
- operation modes
  - `-z` is the default and compresses
  - `-d` decompresses
- operation modifiers
  - `-3` is the default compression level (valid levels are 1 to 19)
  - `--fast` is the ultra-fast compression level
  - `-T0` uses all cpu cores
  - `--long` uses long window
  - `-f` disables many checks
    - input can be symlink, blk, tty
    - output can be tty, and can overwrite existing file
  - `-c` force-outputs to stdout
  - `-o <file>` outputs to the specified file
  - `--sparse`/`--no-sparse` enables/disables sparse (default on unless tty)
- ssh examples
  - `time ssh <remote> "zstd -c <file>" | zstd -dfo <file>`
    - `--no-sparse` if the local file is blk
  - `time zstd -c <file> | ssh <remote> "zstd -dfo <file>"`

## Algorithms and Formats

- compression process
  - dictionary coding
    - when a sequence of bytes matches an entry in the dictionary, replace the
      sequence of bytes by a reference to the dictionary
    - in LZ77, the dictionary is a circular buffer of the past N bytes
  - entropy coding
    - e.g., a text of length `n` requires `7*n` bits in ascii encoding.  If we
      know the frequencies each character appears in the text, we can use a
      different encoding to minimize the bits required to encode the text
    - Huffman can find the optimal encoding in this example
- history of algorithms
  - 1952, Huffman
  - 1977, LZ77
  - 1978, LZ78
  - 1982, LZSS, based on LZ77
  - 1984, LZW, based on LZ78
    - patented and patent expired on 2003
  - 1992, DEFLATE, based on LZ77 and Huffman
  - 1996, LZO, based on LZ77 (and no entropy encoding)
  - 1999, LZMA, based on LZ77
  - 2009, ANS
  - 2011, LZ4, based on LZ77 (and no entropy encoding)
  - 2016, Zstandard, based on LZ77 and ANS
- history of compression formats
  - LZW-based
    - 1985, arc, proprietary
    - 1985, compress, discontinued due to patent issues
    - 1987, gif
  - LZSS-based
    - 1990, arj, proprietary
    - 1993, rar, proprietary
  - DEFLATE-based
    - 1992, pkzip 1.93a, proprietary
    - 1992, info-zip unzip 5.0
    - 1992, gzip and zlib
    - 1996, png
  - LZMA-based
    - 1999, 7z
    - 2010, xz 5.0
      - it was called lzma before 5.0
  - others
    - 1996, bzip2
    - 2016, zstd

## Comparisons

- compressor defaults
  - `bzip2` defaults to `-9`
  - `gzip` defaults to `-6`
  - `lzo` defaults to `LZO1X-1`
  - `lz4` defaults to `-1`
  - `xz` defaults to `-6`
  - `zstd` defaults to `-3`
- <https://github.com/inikep/lzbench>

    | Compressor name       | Compression | Decompress. | Compr. size | Ratio |
    | ---------------       | ----------- | ----------- | ----------- | ----- |
    | xz 5.6.3 -6           | 2.97 MB/s   | 127 MB/s    | 49195929    | 23.21 |
    | zstd 1.5.6 -18        | 3.79 MB/s   | 1169 MB/s   | 53329873    | 25.16 |
    | bzip2 1.0.8 -9        | 13.1 MB/s   | 37.5 MB/s   | 54572811    | 25.75 |
    | zstd 1.5.6 -15        | 8.36 MB/s   | 1369 MB/s   | 57168834    | 26.97 |
    | zstd 1.5.6 -5         | 125 MB/s    | 1197 MB/s   | 63040310    | 29.74 |
    | zlib 1.3.1 -6         | 25.3 MB/s   | 344 MB/s    | 68228431    | 32.19 |
    | zlib-ng 2.2.3 -6      | 62.1 MB/s   | 509 MB/s    | 68861129    | 32.49 |
    | zstd 1.5.6 -2         | 344 MB/s    | 1246 MB/s   | 69503444    | 32.79 |
    | zstd_fast 1.5.6 --1   | 459 MB/s    | 1717 MB/s   | 86984009    | 41.04 |
    | lzo1x 2.10 -1         | 513 MB/s    | 696 MB/s    | 100572537   | 47.45 |
    | lz4 1.10.0            | 577 MB/s    | 3716 MB/s   | 100880800   | 47.60 |

## cpio

- `cpio --format` supports
  - `bin`, defualt but obsoleted, magic `070707` (octal)
  - `odc`, old POSIX, magic `070707` (string)
  - `newc`, cur POSIX, magic `070701` (string)
  - `crc`, cur POSIX with crc, magic `070702` (string)
  - tar
  - other obsoleted formats
- `newc`/`crc` format
  - each file is
    - offset 0: magic
    - offset 6: `st_ino`
    - offset 14: `st_mode`
    - offset 22: `st_uid`
    - offset 30: `st_gid`
    - offset 38: `st_nlink`
    - offset 46: `st_mtim`
    - offset 54: `st_size`
    - offset 62: `major(st_dev)`
    - offset 70: `minor(st_dev)`
    - offset 78: `major(st_rdev)`
    - offset 86: `minor(st_rdev)`
    - offset 94: path size
    - offset 102: checksum or 0
    - offset 110: `\0`-terminated path
    - pad to 32-bit with `\0`
    - data
    - pad to 32-bit with `\0`
  - a final entry with path `TRAILER!!!`

## tar

- `tar --format` supports
  - `gnu`, based on posix 1988
    - this is the default
  - `ustar`, defined in posix 1988
  - `posix` or `pax`, defined in posix 2001
  - other legacy formats
- <https://www.gnu.org/software/tar/manual/html_node/Standard.html>
  - an archive consists of a series of 512-byte blocks
  - a file entry consists of
    - a block for header
      - all fields are strings and are null-terminated
        - numeric values are converted to zero-filled octal numbers
        - e.g., value 10 for a field of width 8 becomes `0000012\0`
      - `char name[100];     /*   0 */`
      - `char mode[8];       /* 100 */`
      - `char uid[8];        /* 108 */`
      - `char gid[8];        /* 116 */`
      - `char size[12];      /* 124 */` is the size of the contents
      - `char mtime[12];     /* 136 */`
      - `char chksum[8];     /* 148 */` is the sum of all bytes in the header
      - `char typeflag;      /* 156 */`
        - `0` is regular file
        - `1` is link
        - `3` is character special
        - `4` is block special
        - `5` is directory
        - `6` is FIFO special
      - `char linkname[100]; /* 157 */`
      - `char magic[6];      /* 257 */` is `ustar `
      - `char version[2];    /* 263 */` is ` `
      - `char uname[32];     /* 265 */` is the user name
      - `char gname[32];     /* 297 */` is the group name
      - `char devmajor[8];   /* 329 */` is major if dev
      - `char devminor[8];   /* 337 */` is minor if dev
      - `char prefix[155];   /* 345 */`
    - zero or more blocks for contents
  - an end-of-archive entry consists of 2 blocks of zeros
  - tar reads/writes N blocks at a time
    - N is 20 by default and the minimum tarball size is thus 10240
    - `-b` can set N

## gzip

- <https://datatracker.ietf.org/doc/html/rfc1952>
  - header
    - `ID1` and `ID2` are 0x1f and 0x8b
    - `CM`, compression method, is typically 8 which denotes deflate
    - `FLG` is
      - bit 0, `FTEXT`
      - bit 1, `FHCRC`
      - bit 2, `FEXTRA`
      - bit 3, `FNAME`
      - bit 4, `FCOMMENT`
    - `MTIME` is the 4-byte modification time of the original file
    - `XFL` is extra flags
    - `OS` is os/fs
      - 3 is UNIX
  - if `FNAME`, zero-terminated name of the orignal file
  - if `FCOMMENT`, zero-terminated comment
  - deflate-compressed data
    - <https://datatracker.ietf.org/doc/html/rfc1951>
  - footer
    - `CRC32` is the crc32 of the original file
    - `ISIZE` is the size of the original file mod 2^32
- <https://datatracker.ietf.org/doc/html/rfc1950>
  - header
    - `CMF`
      - bit 0..3: CM, typically 8 to denote deflate
      - bit 0..3: CINFO, log2 of window size
    - `FLG`
      - bit 0..4: FCHECK, a checksum
      - bit 5: FDICT, preset dictionary
      - bit 6..7: FLEVEL, compression level
    - if `FLG.FDICT`, `DICTID`
  - deflate-compressed data
  - footer
    - `ADLER32` checksum
- http `Content-Encoding`
  - `gzip` denotes gzip format
  - `deflate` denotes zlib format
  - unlike zip, both formats do not require random access and can be used for
    streaming

## zip

- <https://en.wikipedia.org/wiki/ZIP_(file_format)>
  - <https://pkware.cachefly.net/webdocs/APPNOTE/APPNOTE-6.3.10.TXT>
  - End of central directory (EOCD) record at the end
    - byte 0: `PK\5\6`
    - byte 4: Number of this disk, 0 unless multi-part
    - byte 6: Disk where central directory starts, 0 unless multi-part
    - byte 8: Number of central directory records on this disk
    - byte 10: Total number of central directory records
    - byte 12: Size of central directory (bytes)
    - byte 16: Offset of start of central directory
    - byte 20: Comment length
    - byte 22: Comment
      - because of arbitrary comment, must scan backward to find the start of
        EOCD
  - each file has a Central directory file header (CDFH)
    - byte 0: `PK\1\2`
    - byte 4: Version made by
      - the higher byte is the compat os
        - 0 for dos
      - the lower byte is the compat zip spec version times 10
        - 2.0: folder, deflate
        - 4.5: zip64
        - 4.6: bzip
        - 6.3: lzma
    - byte 6: Version needed to extract (minimum)
    - byte 8: General purpose bit flag
    - byte 10: Compression method
      - 0 for stored
      - 8 for deflate
      - 12 for bzip
      - 14 for lzma
      - 93 for zstd
      - 95 for xz
    - byte 12: File last modification time
      - bit 0..4: day (1-31)
      - bit 5..8: month (1-12)
      - bit 9..15: year (relative to 1980)
    - byte 14: File last modification date
      - bit 0..4: second divided by 2 (0..29)
      - bit 5..10: minute (0-59)
      - bit 11..15: hour (0-23)
    - byte 16: CRC-32 of uncompressed data
    - byte 20: Compressed size
    - byte 24: Uncompressed size
    - byte 28: File name length (n)
    - byte 30: Extra field length (m)
    - byte 32: File comment length (k)
    - byte 34: Disk number where file starts, 0 unless multi-part
    - byte 36: Internal file attributes
    - byte 38: External file attributes
    - byte 42: Relative offset of local file header
    - byte 46: File name
    - byte 46+n: Extra field
      - each field consists of header and data
      - each header consists of 2-byte header id and 2-byte data size
        - 0x5455 is extended timestamp (mtime/atime/ctime in epoch)
    - byte 46+n+m: File comment
  - each file has a local file header (for redundancy) followed by the
    compressed data
    - byte 0: `PK\3\4`
    - byte 4: Version needed to extract (minimum)
    - byte 6: General purpose bit flag
    - byte 8: Compression method
    - byte 10: File last modification time in DOS format
    - byte 12: File last modification date in DOS format
    - byte 14: CRC-32 of uncompressed data
    - byte 18: Compressed size
    - byte 22: Uncompressed size
    - byte 26: File name length (n)
    - byte 28: Extra field length
    - byte 30: File name
    - byte 30+n: Extra field
