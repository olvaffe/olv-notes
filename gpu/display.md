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

## Planes
