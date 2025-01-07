Data Manipulation
=================

## Data Manipulations

- compression
  - lossless compression
  - lossy compression
- cryptography
  - symmetric-key cryptography
  - public-key (asymmetric) cryptography
- hash function
  - non-cryptographic
  - cryptographic

## Compression

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

## Data Compressions

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

## Video Compressions

- MPEG
  - MPEG-1 part-2
    - 1993
    - used by VCD
    - ffmpeg
  - MPEG-2 part-2 / H.262
    - 1995
    - used by DVD, Blu-ray, SDTV (DVB, ATSC)
    - ffmpeg
  - MPEG-4 part-2
    - 1999
    - used by DivX, Xvid
    - xvid, ffmpeg
  - MPEG-4 part-10 AVC / H.264
    - 2003
    - used by Blu-ray, HDTV/UHDTV (DVB, ATSC), streaming
    - x264, ffmpeg
  - MPEG-H Part 2 HEVC / H.265
    - 2013
    - used by UHD Blu-ray, UHDTV
    - x265, ffmpeg
  - MPEG-I Part 3 VVC / H.266
    - 2020
    - used by ???
    - vvenc, vvdec
- Microsoft
  - VC-1
    - 2006
    - used by some Blu-ray, WMV, streaming
    - ffmpeg
- Google
  - VP8
    - 2008
    - used by streaming
    - libvpx, ffmpeg
  - VP9
    - 2013
    - used by streaming
    - libvpx, ffmpeg
- AOMedia
  - AV1
    - 2018
    - used by streaming
    - svt-av1, dav1d, dav1d, libaom-av1, ffmpeg

## Audio Compressions

- MPEG
  - MPEG-1 (and 2) part-3 Layer III / MP3
    - 1993
    - ffmpeg
  - MPEG-2 Part 7 AAC
    - 1997
    - ffmpeg, fdk-aac
  - MPEG-4 Part 3 HE-AAC
    - 2003
    - ffmpeg, fdk-aac
  - MPEG-D Part 3 xHE-AAC
    - 2012
    - fdk-aac
- Dolby
  - AC-3
    - 1991
    - ffmpeg
  - E-AC-3
    - 2012
    - ffmpeg
  - AC-4
    - 2015
- Apple
  - ALAC
    - 2004
    - used by Apple
    - ffmpeg
- Xiph
  - Vorbis
    - 2000
    - used by streaming
    - libvorbis, ffmpeg
  - FLAC
    - 2001
    - libflac, ffmpeg
  - Opus
    - 2012
    - libopus, ffmpeg

## Image Compressions

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

## Cryptography

- common encryption algorithms
  - symmetric
    - DES, 1977
    - Blowfish, 1993
    - AES, 2001
    - ChaCha20, 2008
  - asymmetric
    - RSA, 1977
    - DSA, 1994
    - ECDSA, 2000
    - EdDSA, 2011 (ED25519)
- key exchange algorithms
  - DH, 1976 (Diffie–Hellman)
    - A and B publicly agree `x` and `p`
    - A picks `a` privately
    - B picks `b` privately
    - A sends the result of `x^a mod p` to B
    - B sends the result of `x^b mod p` to A
    - both A and B use the result of `x^(ab) mod p` as the secret key
  - ECDH (Elliptic-curve Diffie–Hellman)

## Hash Functions

- common hash functions
  - non-cryptographic
    - CRC-32
    - FNV-1a
    - SipHash, 2012
    - MurmurHash3, 2012
    - xxHash3, 2019
  - cryptographic
    - MD5, 1992
    - SHA-1, 1995
    - SHA-2, 2001 (SHA-256, SHA-512, etc.)
    - SHA-3, 2016
    - BLAKE3, 2020
