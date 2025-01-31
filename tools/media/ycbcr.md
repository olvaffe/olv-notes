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

## Gamma correction

- for a game asset,
  - the stored color values are gamma-corrected
    - that is, linear color values go through OETF before storage
  - the stored color values can be fed to the display directly
    - that is, the display applies EOTF to get linear light intensity
  - the game can choose between
    - read and output the gamma-corrected values without processing, or with
      incorrect processing
    - read the gamma-corrected values, applies EOTF, process in the linear
      space, applies OETF, and output the gamma-corrected values

## CIE 1931

- color matching functions
  - experiment setup
    - a screen that is partitoned into left and right
    - the left half is lit with a test light
    - the right half is lit with a red, a green, and a blue light
    - observer observes with a 2 degree angle of view
    - observer uses 4 knobs to adjust the intensities of the 4 lights, until the
      left and the right match
  - the experiement tells us
    - some amount of test light equals to a certain mixture of r/g/b lights
  - after normalization, we get
    - `lambda = r(lambda) + g(lambda) + b(lambda)`
    - `lambda` is the wavelength of the test light
    - `r(lambda)` is the intensities of the red light after normalization
    - `g(lambda`) and `b(lambda)` are similar
  - repeating the same experiement for different test lights, we get three
    color matching functions `r`, `g`, and `b` 
    - these functions can have negative values at certain wavelengths, meaning
      the only way to match those wavelengths in the experiments was to light
      the left half of the screen with some amount of the r/g/b lights
  - controls
    - the wavelengths of the r/g/b lights
    - using NPL standard,
      - red is 700nm
      - green is 546.1nm
      - blue is 435.8nm
- luminous efficiency function
  - similar to the color matching functions, we can design an experiment to
    learn the perceived luminance of a wavelength
  - this gives us a `y(lambda)`
    - note that this is relative and is normalized to `[0, 1]`
- XYZ color space
  - we can pick any r/g/b light in the color matching experiment
    - because we can color match the new primaries using the original color
      matching functions to get a 3x3 matrix
    - the new color matching functions can be derived from the original ones
      by applying the 3x3 matrix
  - for convenience, we can pick r/g/b lights such that
    - their color matching functions are non-negative
    - one of lights represents luminance
    - more preferable properties
    - note that these lights are imaginary and do not physically exist
  - this gives us unnormalized XYZ color space
    - this color space is additive
    - that is, we can add two colors component-wise
  - and the normalized xyY color space
    - `x = X / (X + Y + Z)`
    - `y = Y / (X + Y + Z)`
    - this color space is not additive
  - to convert xyY back to XYZ,
    - `X = Y * x / y`
    - `Z = Y * (1 - x - y) / y`
- define a new color space
  - it is common to define a color space by defining its primaries and its
    white point in only xy
  - this allows xyY to be calculated
    - Y is 1 for the white point
    - using the equation above to convert xy to XZ, we can solve Y for the 3
      primaries
  - can we define a color space by defining its primaries in XYZ?
    - yes, and the primaries can be summed together to get the white point
    - but it is actually more natural to specify the color space using the
      chrominances of the primaries and the white point, and derive their
      luminances
      - this allows the white point to vary
      - the white point is also the ambient light environment and being able
        to vary it can be handy
- to convert from one color space to another,
  - apply a matrix to convert from the src color space to XYZ color space
  - apply a matrix for chromatic adaption if the white points of the two color
    spaces differ
    - also known as white balance in the context of a camera
    - <http://yuhaozhu.com/blog/chromatic-adaptation.html>
  - apply a matrix to convert from XYZ color space to the dst color space
- <https://medium.com/hipster-color-science/a-beginners-guide-to-colorimetry-401f1830b65a>
- a color is a mix of lights of different wavelengths
- it is not possible to reproduce a color, until we consider the limits of our
  eyes
- our eyes have three cones.  They are most sensitive to long wavelengths (R),
  middle wavelengths (G), and short wavelengths (G) respectively.  They have
  low enough resolutions that we can reproduce the perceived color by using a
  different mix of lights.
- two lights of same luminance (cd/m^2) but different wavelengths can have
  different brightness
- CIE 1931 has three functions called RGB color matching functions.  They are
  "matching" functions in that they matches the perceived color using three
  reference lights.
  - r\_bar(lambda), g\_bar(lambda), and b\_bar(lambda) specify how to reproduce a light of
    wavelength using reference r, g, and b lights.  They are given through
    experiments.
  - R(lambda) = r\_bar(lambda) / 1.0
    G(lambda) = g\_bar(lambda) / 4.5907
    B(lambda) = b\_bar(lambda) / 0.0601

    to scale from brightness of lights to luminance of lights (????)
  - r = R / (R + G + B), g = G / (R + G + B), b = 1 - r - g, after discarding
    intensity information
  - now we have a curve on the r-g plane
  - (1/3, 1/3) is the white
