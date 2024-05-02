Font
====

## Popular Open Fonts

- Noto
  - best unicode coverage
  - default on Fedora 36
- DejaVu
  - derived from Bitstream Vera
  - added more unicode coverage
- Roboto
  - default font on Android
  - older Android used Droid
- Ubuntu
  - default font on Ubuntu
- Liberation
  - metric-compatible with Times New Roman, Arial, Courier New
- for coding
  - Anonymous Pro
  - Inconsolata
  - Source Code Pro

## DPI

- in publishing, common font sizes are 10pt, 11pt, or 12pt
  - 1pt is 1/72 inch
  - latex defaults to 10pt
  - in home printing, some like to add 1pt because home printers used to have
    lower resolution (~300dpi)
- I like 10pt on my laptop screens
  - it is compact which is more suitable for small laptop screens
  - on external displays, 11pt is better
    - 11pt is more comfortable
    - the viewing distance is also larger
- common dpis
  - 14" FHD (1920x1080) laptops or 27" 4K (3840x2160) external displays are
    ~160dpi
  - 14" 4K laptops are ~320dpi
  - android definitions
    - `mdpi` is 160 and is the baseline
    - `ldpi` is 120
    - `hdpi` is 240
- softwares hard-code 96dpi
  - when `s` pt is requested, they pick `s * 96 / 72 * scale` px
  - 96dpi came from windows
    - it made text one-third larger to accomodate for lower display dpi and
      larger viewing distance
  - `scale` is determined by compositors
    - a common formula is `scale = floor(real_dpi / 96)`
    - dpis equal to or larger than `96 * 2` are called HiDPI
  - `scale` is an integer because it applies to text and ui elements
    - fonts are vectors and support fractional scales
    - ui elements tend to be bitmaps and look blurry with fractional scales
    - the scale is also called "ui scale", "device scale", etc.
  - toolkits are slowly gaining fractional scale support for ui elements
- we really need fractional scale for 160dpi
  - without it, UI elements are small but OK
    - because 160dpi is common enough that designers have accomodated for it
  - without it, text is way too small on linux
    - 10pt should be `10 * 160 / 72 = 22.2`px
    - but softwares pick `10 * 96 / 72 = 13.3`px
    - scaling text and ui elements by 2 makes text readable, but it also makes
      ui elements too huge
- Wayland
  - there is (integral) output scale
  - there is now the newer fractional scale, `wp_fractional_scale_v1`
- each app requires a different solution

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
  - this scales the text and is an accessibility setting
- there is `GDK_SCALE` that applies an integral scale to both the ui and the
  text, but it is x11-only
- there was also `GDK_DPI_SCALE` that aplpies a fractional scale to just the
  text which has been removed

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
