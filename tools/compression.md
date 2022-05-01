Compressions
============

## Data

- using numbers from lzbench
- traditionally, best ratio, balanced, and best speed compressors are

| Compressor name         | Compress.  |Decompress. | Compr. size | Ratio |
| ---------------         | -----------| -----------| ----------- | ----- |
| bzip2 1.0.8 -9          |    15 MB/s |    41 MB/s |    54572811 | 25.75 |
| zlib 1.2.11 -6          |    35 MB/s |   407 MB/s |    68228431 | 32.19 |
| lzo1x 2.10 -11          |   735 MB/s |   893 MB/s |   106604629 | 50.30 |

- but things have changed
- best ratio

| xz 5.2.4 -9             |  2.62 MB/s |    88 MB/s |    48745306 | 23.00 |
| xz 5.2.4 -6             |  2.95 MB/s |    89 MB/s |    49195929 | 23.21 |
| zstd 1.4.3 -22          |  2.28 MB/s |   865 MB/s |    52738312 | 24.88 |
| zstd 1.4.3 -18          |  3.58 MB/s |   912 MB/s |    53690572 | 25.33 |
| bzip2 1.0.8 -9          |    15 MB/s |    41 MB/s |    54572811 | 25.75 |

- balanced

| zstd 1.4.3 -5           |   104 MB/s |   932 MB/s |    63993747 | 30.19 |
| zstd 1.4.3 -2           |   356 MB/s |  1067 MB/s |    69594511 | 32.84 |
| zlib 1.2.11 -6          |    35 MB/s |   407 MB/s |    68228431 | 32.19 |

- best speed

| lz4 1.9.2               |   737 MB/s |  4448 MB/s |   100880800 | 47.60 |
| lzo1x 2.10 -11          |   735 MB/s |   893 MB/s |   106604629 | 50.30 |

## Videos

- MPEG
  - MPEG-1 part-2
    - 1993
    - used by VCD
  - MPEG-2 part-2 / H.262
    - 1995
    - used by DVD, Blu-ray, SDTV (DVB, ATSC)
  - MPEG-4 part-2
    - 1999
    - used by DivX, Xvid
  - MPEG-4 part-10 AVC / H.264
    - 2003
    - used by Blu-ray, HDTV/UHDTV (DVB, ATSC), streaming
  - MPEG-H Part 2 HEVC / H.265
    - 2013
    - used by UHD Blu-ray, UHDTV
  - MPEG-I Part 3 VVC / H.266
    - 2020
    - used by ???
- Microsoft
  - VC-1
    - 2006
    - used by some Blu-ray, WMV, streaming
- Google
  - VP8
    - 2008
    - used by streaming
  - VP9
    - 2013
    - used by streaming
- AOMedia
  - AV1
    - 2018
    - used by streaming

## Audio

- MPEG
  - MPEG-1 (and 2) part-3 Layer III / MP3
    - 1993
  - MPEG-2 Part 7 AAC
    - 1997
  - MPEG-4 Part 3 HE-AAC
    - 2003
  - MPEG-H Part 3 3D Audio
    - 2015
- Dolby
  - AC-3
    - 1991
  - E-AC-3
    - 2012
  - AC-4
    - 2015
- Apple
  - ALAC
    - 2004
    - used by Apple
- Xiph
  - Vorbis
    - 2000
    - used by streaming
  - FLAC
    - 2001
  - Opus
    - 2012

## Images

- GIF
  - 1992
  - used for animation
- JPEG
  - 1992
  - used everywhere
- PNG
  - 1996
  - used for lossless
- AVC in HEIF
  - 2003
  - no user?
- WebP
  - 2011
  - used for web
- HEVC in HEIF
  - 2013
  - used by Apple
- AV1 in HEIF
  - 2019
  - no user?
- JPEG-XL
  - 2021
  - lossy outperforms AV1 / HEVC / WebP
  - lossless outperforms WebP / PNG
