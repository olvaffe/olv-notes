Color Management
================

## CIE 1931

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
