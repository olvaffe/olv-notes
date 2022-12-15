Color Management
================

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

## Wide Color Display
