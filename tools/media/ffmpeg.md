FFmpeg
======

## Overview

- <https://git.ffmpeg.org/ffmpeg.git>
- libraries
  - `libavutil` provides string functions, rng, math, etc.
  - `libswscale` provides image scaling, color space and format conversions,
    etc.
  - `libswresample` provides audio resampling, format conversion, and
    rematrixing
  - `libavcodec` provides decoders and encoders (codecs) for audio, video,
    and subtitle streams
    - `ffmpeg -codecs` lists them
  - `libavformat` provides demuxers and muxers for audio, video, and subtitle
    streams
    - it also supports several input and output protocols
    - `ffmpeg -formats` and `ffmpeg -protocols` list them
  - `libavdevice` provides input/output devices such as v4l2, alsa, opengl, etc.
      - `ffmpeg -devices` list them
  - `libavfilter` provides filters for audio and video streams
      - `ffmpeg -filters` list them
- `ffmpeg <options> <inputs> <outputs>`
  - `<options>` are global options
  - `<inputs>` are arbitrary number of inputs and input options
  - `<outputs>` are arbitrary number of outputs and output options
  - input/output options apply to the following input/output
- global options
  - `ffmpeg -v debug` to show debug messages
- for each input,
  - there must be protocol support to read from the input
    - `ffmpeg -protocols` to show supported protocols
    - `ffmpeg -i file://url` to force `file` protocol
  - there must be format support to demux the input into streams
    - `ffmpeg -formats` shows the supported formats
      - `ffmpeg -demuxers` shows the supported formats that are demuxers
      - `ffmpeg -devices` shows the supported formats that are devices
    - `ffmpeg -f mp4` to force `mp4` format
  - there must be codec support for each stream
    - `ffmpeg -c copy` can copy a stream without the need for a codec
    - `ffmpeg -codecs` shows the supported codecs
      - `ffmpeg -decoders` shows the supported codec decoders
      - a codec might have multiple decoders
        - `av1` has `libdav1d`, `libaom-av1`, etc.
    - `ffmpeg -c:v libdav1d` to force `libdav1d` decoder for all video streams
- for each output,
  - it is similar to an input
  - `ffmpeg -muxers` shows the supported formats that are muxers
  - `ffmpeg -encoders` shows the supported codec encoders
- stream specifier
  - `-codec ac3` selects ac3 for all streams
  - `-codec:a:0 ac3` selects ac3 for audio stream #0

## Pixel Formats

- let `a.pnm` be a 64x64 image with solid color `#aabbcc`
  - assuming bt709 and limited range, it is `(0xae, 0x8a, 0x77)` in yuv
- `ffmpeg -i a.pnm -pix_fmt <fmt> a.rgb`
  - `rgb24` outputs `aabbcc` of size `64*64*3`
  - `argb` outputs `ffaabbcc` of size `64*64*4`
- `ffmpeg -i a.pnm -pix_fmt <fmt> a.yuv`
  - `nv12` outputs `ae` for Y and `8a77` for UV of size `64*64*1.5`
  - `yuv420p` outputs `ae` for Y, `8a` for U, and `77` for V of size
    `64*64*1.5`
  - `yuv420p` outputs `ae` for Y, `8a` for U, and `77` for V of size
    `64*64*1.5`
    - p stands for planar

## Hardware Acceleration

- `ffmpeg -hwaccels` shows the supported hwaccel methods

## Multimedia Players

- multimedia players sorted by popularity
  - vlc
    - uses ffmpeg and others directly
    - provides `libvlc` for embedding
  - mpv
    - uses ffmpeg and others directly
    - provides `libmpv` for embedding
  - totem
    - uses gstreamer
  - dragon
    - uses phonon
- multimedia frameworks
  - gstreamer
    - uses ffmpeg and others directly
  - phonon
    - uses gstreamer, vlc, and others
- hardware abstraction
  - pipewire abstracts
    - alsa for audio record/playback
    - v4l2 for video record (and playback?)
    - display for display recording
  - ffmpeg does not use pipewire yet
    - uses alsa or pulseaudio for audio
    - uses v4l2 directly
    - uses x11grab for display recording

## libavformat

- config.mak decides which files to compile
  config.h helps decide what goes into allformats.c
- mplayer -lavfdopts format=help <somefile> to list supported formats

## libavcodec

- config.mak decides which files to compile
  config.h helps decide what goes into allcodecs.c

## ffmpeg encoding

- `libavformat/output-example.c`
- one buffer is allocated for AVFrame, another buffer is allocated for
  compressed data
- `avcodec_encode_video` encodes a frame and outputs to the second buffer.

## Blu-ray

- DRM
  - the player has a host key and a host cert
  - the player reads MKB (media key block) from the disk, decrypts it using
    host key, and gets the media key
  - the player presents the host cert to read VID (volume id) from BD-ROM mark
    region
  - the player generates VUK (volume unique key) from media key and VID
  - the player reads title keys, decrypts them using VUK
  - the player decrypts contents using title keys
- native video formats
  - the most common format is 1920x1080, 24fps, 8-bit
    - 4K is 3840x2160, 24fps, 10-bit
  - it can be h262, h264, vc1
    - 4K is h265
- native audio formats
  - the most common format is 48khz, 5.1ch, 24-bit
    - it can go up to 96khz or 192khz, but rare
    - it can go up to 7.1ch
  - it can be uncompressed (pcm/lpcm), dolby truehd (ac3), dts-hd (dts)
- transcode video formats
  - ideally, keep the native resolution and fps
    - if optimizing for size, `-filter:v scale=-1:720` to downsize to 720p
    - `-pix_fmt yuv420p10le` to use 10-bit to avoid banding
  - av1 for quality and open
  - h264 for compat
    - e.g., `-pix_fmt yuv420p10le -filter:v scale=-1:720 -c:v libx264 -preset slow -crf 23`
  - to burn in image-based subtitle, `-filter_complex '[0:0][0:2]overlay[v]' -map '[v]'`
    - that is, overlay stream 0 and stream 2
- transcode audio formats
  - ideally, keep the native sample rate, sample depth, and channels
    - if optimizing for size, `-ac 2` to downmix to 2 channels
    - no point in changing the sample rate/depth
  - opus for quality and open
  - aac for compat, targeting 64kbs/ch
    - e.g., `-ac 2 -c:a aac -b:a 128k`
- for casual trancode with best compat,
  - container: mp4, which lacks subtitle support
  - video codec: h264, 8-bit
  - audio codec: aac
  - assuming stream 0 is video, stream 1 is audio, and stream 2 is image-based
    subtitle
  - `-filter_complex '[0:0][0:2]overlay,scale=-1:720'` burns in subtitle and
    resize to 720p
  - `-map 0:1` picks the audio track
  - `-c:v libx264 -preset slow -crf 23`
  - `-c:a aac -ac 2 -b:a 128k`
- for archival trancode with open formats,
  - container: mkv
  - video codec: av1, 10-bit
  - audio codec: opus
  - `-c:v libsvtav1 -crf 20 -preset 4 -pix_fmt yuv420p10le`
    - <https://gitlab.com/AOMediaCodec/SVT-AV1/-/blob/master/Docs/CommonQuestions.md#what-presets-do>
    - <https://gitlab.com/AOMediaCodec/SVT-AV1/-/blob/master/Docs/CommonQuestions.md#8-or-10-bit-encoding>
  - `-c:a opus -b:a 128k` for stereo, `256k` for 5.1, `450k` for 7.1
    - <https://wiki.xiph.org/Opus_Recommended_Settings>

## DVD-Video

- <https://en.wikipedia.org/wiki/DVD-Video>
  - the fs is UDF
  - `AUDIO_TS` is empty
    - it is for DVD-Audio
  - `VIDEO_TS`
    - VMG (video manager) files
      - `VIDEO_TS.IFO` is the metadata for the disc
      - `VIDEO_TS.BUP` is the backup of `VIDEO_TS.IFO`
      - `VIDEO_TS.VOB` is the first-play object, usually copyright notice or
    - multiple VTS (video title set) files
      - `VTS_mm_0.IFO` is the metadata for VTS `mm`
        - `mm` ranges from 01 to 99
      - `VTS_mm_0.BUP` is the backup of `VTS_mm_0.IFO`
      - `VTS_mm_n.VOB` are the video objects for VTS `mm`
        - `0` is the menu
        - `1` or later is the video
          - DVD-Video requires each VOB file to be less than 1GB, and as such,
            the video is splitted into multiple VOB files
      - for a tv show, each `mm` typically corresponds to an episode
- <https://en.wikipedia.org/wiki/VOB>
  - VOB is a subset of MPEG-PS container
    - PS stands for program stream and uses `.mpg`, `.mpeg`, `.m2p`, or `ps`
      as the suffix
  - reading most VOBs result in I/O error
    - `[sr0] tag#0 Sense Key : Illegal Request [current]`
    - `[sr0] tag#0 Add. Sense: Read of scrambled sector without authentication`
  - the video is typically `mpeg2video (Main), yuv420p(tv, progressive), 720x480 [SAR 32:27 DAR 16:9], 29.97 fps`
  - the audio is typically `ac3, 48000 Hz, 5.1(side), fltp, 384 kb/s`
- <https://en.wikipedia.org/wiki/Content_Scramble_System>
  - there are 3 participants: disc, drive, and player
  - there are 3 protection methods
    - the drive may deny access if the disc has a different region code
      - `regionset` can change the drive region (usually up to 5 times)
        - it uses cdrom `DVD_AUTH` ioctl
    - the drive may deny access to certain disk regions unless the player
      authenticates
    - the data may be encrypted and the player need to decrypt
- <https://en.wikipedia.org/wiki/Libdvdcss>
  - due to legal concerns, player typically uses `libdvdread` to access the
    dvd
  - when `libdvdcss` exists, `libdvdread` uses it to decrypt
- for casual trancode with best compat,
  `ffmpeg -i 'concat:<input1>.vob|<input2>.vob' -filter_complex '[0:1][0:7]overlay' -c:v libx264 -preset slow -crf 19 -c:a aac -ac 2 -b:a 128k <output>.mp4`
  - `-filter_complex '[0:1][0:7]overlay'` overlays stream 7 (subtitle) over
    stream 1 (video)
    - this does not work when stream 7 means different langs for
      `<input1>.vob` and `<input2>.vob`
  - `-ac 2` downmixes to stereo
