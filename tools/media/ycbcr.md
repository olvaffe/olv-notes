YCbCr
=====

## Overview

- <https://en.wikipedia.org/wiki/YCbCr>
- CRT expresses colors in RGB
- but RGB values are not efficient for storage
- Y'CbCr is a coordinate transformation of the underlying RGB, for efficient
  stroage
  - the prime sign means gamma-corrected
- from R'G'B' to Y'PbPr
  - the transformation is defined by `Kr`, `Kg`, and `Kb` coefficients
    - with `Kr+Kg+Kb=1.0`
  - R'G'B' is in `[0.0, 1.0]`
  - `Y' = Kr*R'+Kg*G'+Kb*B'` is in `[0.0, 1.0]` as well
    - each coefficient represents the luma of the channel
  - `Pb` and `Pr` are in `[-0.5, 0.5]`
- from Y'PbPr to Y'CbCr
  - in 8-bit full range, all 3 channels are mapped to `[0, 255]`
    - all channels are scaled by 255 first
    - 128 is then added to Cb and Cr
  - in 8-bit limited range, Y' is mapped to `[16, 235]` and Pb/Pr are mapped
    to `[16, 240]`
    - Y' is scaled by 219 and Pb/Pr are caled by 224 first
    - 16 is then added to Y' and 128 is added to Cb and Cr
  - limited range is more common
- ITU-R BT.601
  - the coefficients are `(0.299, 0.587, 0.114)`
  - this is for SDTV
- ITU-R BT.709
  - the coefficients are `(0.2126, 0.7152, 0.0722)`
  - this is for HDTV and is more common
- subsampling
  - `4:4:4` has no subsampling
  - `4:2:2` are subsampled horizontally
    - `YUY2` or `YUYV` has `y0, u, y1, v`
    - `UYVY` has `u, y0, v, y1`
  - `4:2:0` are subsampled horizontally and vertically
    - `YV12` has Y plane, V plane, and U plane
    - `NV12` has Y plane and UV plane
