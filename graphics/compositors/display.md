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

- DisplayPort 1.0
  - 2006
  - 10.8 Gb/s (HBR x 4 lanes)
  - DPCP (DisplayPort Content Protection)
- DisplayPort 1.1
  - 2007
  - alternative link layers such as fiber optic
  - HDCP (High-bandwidth Digital Content Protection) 1.3
  - DisplayPort Dual-Mode (DP++), allowing DVI and HDMI adapters
  - stereoscopic 3D
- DisplayPort 1.1a
  - 2008
- DisplayPort 1.2
  - 2010
  - 21.6 Gb/s (HBR2 x 4 lanes)
  - 4K at 60 Hz at 10 bpc
  - MST (Multi-Stream Transport)
  - more color spaces (sRGB, scRGB, DCI-P3)
  - compatible with Mini DisplayPort connector
- DisplayPort 1.2a
  - 2013
  - HDCP 2.2
  - optional Adaptive Sync
- DisplayPort 1.3
  - 2014
  - 32.4 Gb/s (HBR3 x 4 lanes)
  - 8K at 30 Hz
  - BT.2020
- DisplayPort 1.4
  - 2016
  - DSC (Display Stream Compression) 1.2
  - HDR10 with static and dynamic metadata
- DisplayPort 1.4a
  - 2018
  - DSC 1.2a
- DisplayPort 2.0
  - 2019
  - 80.0 Gb/s (UHBR 20 x 4 lanes)
- DisplayPort 2.1
  - 2022
- eDP
  - eDP 1.0, 2008
  - eDP 1.1/1.1a, 2009
  - eDP 1.2, 2010
  - eDP 1.3, 2011
    - PSR (Panel Self-Refresh)
  - eDP 1.4, 2013
    - based on DP 1.2
  - eDP 1.4a, 2015
    - based on DP 1.3
  - eDP 1.4b, 2015
  - eDP 1.5, 2021
- DisplayID, an EDID replacement
- Dual Mode
  - when a dp connector supports dual-mode, it internally connects to both a
    dp phy and an hdmi phy
  - the display controller will output dp signal or hdmi signal depending on
    whether the sink is dp or hdmi
  - as such, dp-to-hdmi cable is a passive adapter and is simple

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

## Common Resolutions

- tv standards
  - HD, 1280x720
  - FHD, 1920x1080
  - QHD, 2560x1440
  - 4K UHD, 3840x2160
  - 8K UHD, 7680x4320
- pc standards
  - VGA, 640x480
  - SVGA, 800x600
  - XGA, 1024x768
  - SXGA, 1280x1024
  - UXGA, 1600x1200

## Planes
