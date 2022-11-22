FFmpeg
======

## Overview

- <https://git.ffmpeg.org/ffmpeg.git>

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
