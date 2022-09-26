Font
====

## DPI

- in publishing, common font sizes are 10pt, 11pt, or 12pt
  - 1pt is 1/72 inch
- I like 10pt for its compactness
  - at 96dpi, that's 13.3px
  - at 160dpi, that's 22.2px
  - at 300dpi, that's 41.6px
  - computer standards on 15" monitor
    - VGA, 640x480, is ~55dpi
    - SVGA, 800x600, is ~70dpi
    - XGA, 1024x768, is ~85dpi
    - SXGA, 1280x1024, is ~110dpi
    - UXGA, 1600x1200, is ~135dpi
  - tv standards on 15" monitor
    - HD, 1280x720, is ~100dpi
    - FHD, 1920x1080, is ~150dpi
    - UHD 4K, 3840x2160, is ~300dpi
    - UHD 8K, 7680x4320, is ~600dpi
- many softwares get dpi "wrong"
  - system softwares/libraries commonly assume 96dpi
    - but some detect the real dpi
    - and some assume 72dpi or 75dpi
  - they are not "wrong" though
    - when dpi changes, we want to scale both ui elements and text
    - but dpi applies to text only
    - modern softwares assume fixed 96dpi as the logical dpi
    - they have a separate "scale" (aka "ui scale", "device scale") to scale
      both ui elements and text
  - the problem is that "scale" is integral
    - they are slowly gaining fractoinal scale support
- common scenarios
  - a 14" laptop at 1920x1080 is ~160dpi
    - we want a `160/96 = 1.66` scale
    - only fractional scale works
  - a 32" monitor at 3840x2160 is ~140dpi
    - we want a `140/96 = 1.45` scale
    - only fractional scale works
  - a 14" laptop at 3840x2160 is ~320dpi
    - we want a `320/96 = 3.33` scale
    - 3 can work
    - this is known as HiDPI
- Wayland
  - there is (integral) output scale
  - fractional scale is wip
- at ~150dpi, we really need fractional scale
  - without it, UI elements are a bit small but OK
    - because this is common enough that designers have accomodated for it
  - without it, text is too small on linux
    - because many system softwares/libraries assume 96dpi
      - they assume 10pt is 13.3px while the correct size is 20.8px
    - doubling the sizes of UI elements and fonts makes text comfortable, but
      UI elements become too huge
    - each app might require a different solution

## fontconfig

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

## xft

- `XftFontMatch` is a wrapper of `FcFontMatch`
  - calls `FcConfigSubstitute` for config substitute
  - calls `XftDefaultSubstitute` which is a wrapper to `FcDefaultSubstitute`
    - `_XftDefaultInit` initializes more defaults from `XGetDefault`, X
      resource db
      - `Xft.scale` for fc `scale`
      - `Xft.dpi` for fc `dpi` 
      - and more
    - if `Xft.dpi` is not set, it uses `DisplayHeight() * 25.4 / DisplayHeightMM()`
    - with `scale`, `dpi`, and `size` known, `FcDefaultSubstitute` calculates
      `pixelsize`
- `XftFontOpenPattern` opens a font using `FcPattern`
  - `_XftSetFace` calls `FT_Set_Char_Size`
  - `xsize` and `ysize` are from `pixelsize`
- `XftDrawString8` internally calls `XftGlyphRender`

## pango

- `PangoFontDescription`
  - `pango_font_description_from_string` parses family, size, etc. from the
    string and returns a `PangoFontDescription`
- `PangoLayout`
  - `pango_cairo_create_layout` creates a `PanglLayout` from a `cairo_t`
  - `pango_layout_set_text(layout, str, -1)` sets the string to lay out
  - `pango_layout_set_font_description` sets the font desc
  - `pango_cairo_update_layout` updates pango's internal states
  - `pango_cairo_show_layout` draws the layout
  - `pango_font_map_load_font` is called internally somewhere
    - it calls `pango_fc_font_map_load_font` which calls
      `pango_fc_font_map_load_fontset`
    - `get_scaled_size` calculates `pixelsize` which honors `dpi`
- `pango_cairo_font_map_new` uses `PANGO_TYPE_CAIRO_FC_FONT_MAP` on linux
  - it does not use Xft nor `Xft.dpi`
  - `pango_cairo_context_set_resolution` is used instead
    - otherwise, by default, it is 96dpi

## gnome

- `gsettings set org.gnome.desktop.interface text-scaling-factor 1.5`
  - this is a accessibility setting

## alacritty

- say, `alacritty.yml` specifies `font: size: 10.0`
- alacritty calls `load_font` with `10.0`
  - crossfont calls `get_face` with `10.0` and sets fontconfig `pixelsize` to
    `10.0 * device_pixel_ratio * 96 / 72`
    - that is, it uses hardcoded dpi 96 and forces `pixelsize`
    - I am not sure if this is used or not
- alacritty calls `get_glyph` with `10.0`
  - crossfont uses `10.0 * device_pixel_ratio * 96 / 72` as the size and calls
    `set_char_size` (`FT_Set_Char_Size`)
  - `FT_Set_Char_Size` doc says that dpi is assumed to be 72 when 0 is
    specified
- it is helpless on a non-96dpi monitor
  - manually times 1.6 on the font size
  - `device_pixel_ratio` is 2 for HiDPI

## chrome

- it respects `Xft.dpi`, but not working on wayland
  - I am using `--ozone-platform=wayland --gtk-version=4`
- current solution
  - Settings -> Appearance -> Page zoom -> 150%
