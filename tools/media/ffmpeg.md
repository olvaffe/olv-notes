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
