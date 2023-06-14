Display
=======

## EDID

- 128-byte structure, optionally followed by other 128-byte extensions
- EDID structure, version 1.4
  - byte 0..19: manufacturer id, product code, serial, year, week, EDID ver
  - byte 20..24: basic dispaly params (bit depth, hdmi/dp, monitor size, etc)
  - byte 25..34: xy coordinates of rgbw
  - byte 35..37: supported (outdated) modes
  - byte 38..53: up to eight 2-byte display mode descriptions
  - byte 54..125: four 18-byte timing/display descriptors
  - byte 126: number of extensions
  - byte 127: checksum

## DisplayPort

## HDMI

- HDMI 1.0
  - 2002
  - 165 MHz / 3.96 Gbs
  - 1080p
- HDMI 1.1
  - 2004
  - DVD-Audio
- HDMI 1.2
  - 2005
  - PC-friendly by allowing RGB-only sources
- HDMI 1.2a
  - 2005
  - CEC, allowing control commands between devices
- HDMI 1.3
  - 2006
  - 340 MHz / 8.16 Gbs
  - 1440p
  - deep color (10, 12, and 16 bpc color depths)
  - cables
    - category 1: up to 74.25 MHz
    - category 2: up to 340 MHz
    - HDMI Type C Mini Connector
- HDMI 1.3a
  - 2006
- HDMI 1.4
  - 2009
  - 4K at 30 Hz
  - 3D
  - HEC (HDMI Ethernet Channel), allowing internet sharing
  - Micro HDMI Connector
- HDMI 1.4a
  - 2010
  - more 3D formats
- HDMI 1.4b
  - 2011
- HDMI 2.0
  - 2013
  - 600 MHz / 18.0 Gbs
  - 4K at 60 Hz
  - Rec. 2020 color space
- HDMI 2.0a
  - 2015
  - HDR with static metadata
- HDMI 2.0b
  - 2016
  - HDR10 and HLG
- HDMI 2.1
  - 2017
  - 48.0 Gbs
  - 4K at 120Hz
  - 8K at 60Hz
  - Ultra High Speed Cable
  - Dynamic HDR
  - DSC (Display Stream Compression) 1.2
  - HFR (High Frame Rate)
  - VRR (Variable Refresh Rate)
- HDMI 2.1a
  - 2022
  - SBTM (Source-Based Tone Mapping)

## Planes
