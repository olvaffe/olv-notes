Font
====

## Fontconfig

- `fc-pattern`
  - calls `FcNameParse` to parse a string into a pattern or `FcPatternCreate`
    to create an empty pattern
  - `-c` calls `FcConfigSubstitute` with `FcMatchPattern` on the pattern
  - `-d` calls `FcDefaultSubstitute` on the pattern
    - this adds default values for many components that are missing in the
      pattern
    - `size` defaults to 12.0, `scale` defaults to 1.0, `dpi` defaults to 75
    - if `pixelsize` is not specified, it defaults to `size * scale / 72 * dpi`
    - if `pixelsize` is specified, `size` defaults to
      `pixelsize / dpi * 72 / scale`
  - if elements are specified, calls `FcPatternFilter` to filter in desired
    elements
  - calls `FcPatternPrint` or `FcPatternFormat` to print the pattern
- `fc-match`
  - calls both `FcConfigSubstitute` and `FcDefaultSubstitute` on the pattern
  - calls `FcFontMatch` to return the best matched pattern
    - it finds the font that best matches the user pattern
    - it calls `FcFontRenderPrepare` merge the font and the user pattern
      - the result is automatically passed to `FcConfigSubstituteWithPat` with
      	`FcMatchFont`
  - when `--sort` or `--all` is specified, it calls `FcFontSort` instead
- Language has higher priority than family
  - `fc-match ":lang=us"`
  - Language can be derived from locale
- `FC_DEBUG=1` shows match pattern and best match
  - `FC_DEBUG=1 fc-match monospace`
- `FC_DEBUG=2` shows all fonts and their scores
  - `FC_DEBUG=2 fc-match monospace`

## Alacritty as an example

- say, `alacritty.yml` specifies `font: size: 16.0`
- alacritty calls `load_font` with `16.0`
  - crossfont calls `get_face` with `16.0` and sets fontconfig `pixelsize` to
    `16.0 * device_pixel_ratio * 96 / 72`
    - that is, it uses hardcoded dpi 96 and forces `pixelsize`
    - I am not sure if this is used or not
- alacritty calls `get_glyph` with `16.0`
  - crossfont uses `16.0 * device_pixel_ratio * 96 / 72` as the size and calls
    `set_char_size` (`FT_Set_Char_Size`)
  - `FT_Set_Char_Size` doc says that dpi is assumed to be 72 when 0 is
    specified

## pattern

- Xft.dpi
