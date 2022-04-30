Font
====

## Fontconfig

- Language has higher priority than family
  - `fc-match ":lang=us"`
  - Language can be derived from locale
- `FC_DEBUG=1` shows match pattern and best match
  - `FC_DEBUG=1 fc-match monospace`
- `FC_DEBUG=2` shows all fonts and their scores
  - `FC_DEBUG=2 fc-match monospace`

## pattern

- Xft.dpi
