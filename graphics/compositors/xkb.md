XKB
===

## Background

- originally `X Keyboard Extension`
  - <https://gitlab.freedesktop.org/xorg/proto/xorgproto/-/tree/master/specs/kbproto>
- window system agnostic bits
  - `libxkbcommon` and `libxkbregistry` libraries
    - <https://xkbcommon.org/>
  - `xkeyboard-config` dataset
    - <https://gitlab.freedesktop.org/xkeyboard-config/xkeyboard-config>
- x11-specific bits
  - `libxkbcommon-x11` library
    - also a part of xkbcommon
  - `libxkbfile` library
    - <https://gitlab.freedesktop.org/xorg/lib/libxkbfile>
  - `libxcb-xkb` library
    - <https://gitlab.freedesktop.org/xorg/lib/libxcb>
  - utilities
    - <https://gitlab.freedesktop.org/xorg/app/setxkbmap>
    - <https://gitlab.freedesktop.org/xorg/app/xkbcomp>
    - <https://gitlab.freedesktop.org/xorg/app/xkbevd>
    - <https://gitlab.freedesktop.org/xorg/app/xkbprint>
    - <https://gitlab.freedesktop.org/xorg/app/xkbutils>
- RMLVO and KcCGST
  - RMLOV: Rules, Model, Layout, Variant, Options
  - KcCGST: Keycode, Compat, Geometry, Symbols, Types
  - users use RMLVO
  - xkbcommon internally translates it to and back from KcCGST
  - RMLVO is a lookup key into `xkeyboard-config` database to get KcCGST
- on x11, I guess
  - `setxkbmap` interacts with the server
    - `setxkbmap <RMLVO>` sets the current mapping
    - `setxkbmap -query` queries the current mapping
  - when the server receives an RMLVO, it uses `xkbcommon` and
    `xkeyboard-config` to translate to KcCGST
  - the server also supports XKM (x keymap)
    - when the client and the server are on different machines, their
      `xkeyboard-config` might be at different versions
    - the client can compile RMLVO to XKM and send XKM to the server instead
      - `setxkbmap -print <RMLVO> | xkbcomp -xkm - $DISPLAY`
  - `xkbprint` can take a XKM and output a PostScript file for the keyboard
- `setxkbmap -print -rules evdev -model pc104 -layout us`
  - a rule determines valid MLVO values
    - it should be `evdev` on linux and `base` on others
  - a model determines the physical layout of keys
    - commonly values are `pc104` or `chromebook`
  - a layout determines what the physical keys output
    - country codes such as `us`
  - a variant determines the minor variations
    - `basic` (default), `intl`, `mac`
  - options are for extra options
  - `man xkeyboard-config`

## xkbcommon

- <https://github.com/xkbcommon/libxkbcommon/blob/master/doc/quick-guide.md>
  - the goal is to compile a `xkb_keymap`
  - we can ask the user to provide an RMLVO and

    const struct xkb_rule_names names = { RMLVO };
    xkb_keymap_new_from_names(ctx, &names, XKB_KEYMAP_COMPILE_NO_FLAGS);
  - or, if we are a wayland client, we can get the keymap from the compositor
    as a string and
    `xkb_keymap_new_from_string(ctx, string, XKB_KEYMAP_FORMAT_TEXT_V1, XKB_KEYMAP_COMPILE_NO_FLAGS)`
  - or we can `xkb_keymap_new_from_names(ctx, NULL, XKB_KEYMAP_COMPILE_NO_FLAGS)`
    and rely on defaults
    - rules is from `XKB_DEFAULT_RULES` envvar or compile-time default `evdev`
    - model is from `XKB_DEFAULT_MODEL` envvar or compile-time default `pc105`
    - layout is from `XKB_DEFAULT_LAYOUT` envvar or compile-time default `us`
    - variant is from `XKB_DEFAULT_VARIANT` envvar or compile-time default NULL
    - options is from `XKB_DEFAULT_OPTIONS` envvar or compile-time default NULL
- a wayland client
  - `wl_keyboard::keymap` gives the keymap
  - `wl_keyboard::key` gives the keycode when a key is pressed/released
  - using xkbcommon,
    - `xkb_keymap_new_from_string` to create the keymap
    - `xkb_state_new(keymap)` to create the state
      - it remembers things like which keyboard modifiers or LEDs are active
    - `xkb_state_key_get_one_sym(state, keycode)` to translate keyboard to
      keysym
- `grep -h modifier_map /usr/share/X11/xkb/symbols/{sy,pc} | sort | uniq`
  - `Mod1` is `Alt`
  - `Mod2` is `Num_Lock`
  - `Mod4` is `Super`
- mapping RMLVO to KcCGST
  - assuming RMLVO is `(evdev, pc105, us, NULL, NULL)`
  - `/usr/share/X11/xkb/rules/evdev` defines the `evdev` rules
    - `! model = keycodes` defines model to keycodes mappings
      - `* = evdev` maps `pc105` to `evdev`
    - `! layout = keycodes` define layout to keycodes mappings
      - `* = +aliases(qwerty)` maps `us` to `+aliases(qwerty)`
    - `! model = geometry` defines model to geometry mappings
      - `$pcmodels = pc(%m)` maps `pc105` to `pc(pc105)`
      - `$pcmodels` includes `pc105`
      - `%m` expands to the model
    - `! model layout = symbols` defines model+layout to symbols mappings
      - `* * = pc+%l%(v)` maps `(pc105, us)` to `pc+us`
      - `%l` expands to layout
      - `%(v)` expands to variant enclosed by parentheses, if specified
    - `! model = symbols` defines model to symbols mappings
      - `* = +inet(evdev)` maps `pc105` to `+inet(evdev)`
    - `! model layout = compat` defines model+layout to compat mappings
      - `* * = complete` maps `(pc105, us)` to `complete`
    - `! model = types` defines model to types mappings
      - `* = complete` maps `pc105` to `complete`
  - KcCGST is `(evdev+aliases(qwerty), complete, pc(pc105), pc+us+inet(evdev), complete)`
    - `/usr/share/X11/xkb/keycodes/evdev`
      - `default xkb_keycodes "evdev"`
    - `/usr/share/X11/xkb/keycodes/aliases`
      - `default xkb_keycodes "qwerty"`
    - `/usr/share/X11/xkb/compat/complete`
      - `default xkb_compatibility "complete"`
    - `/usr/share/X11/xkb/geometry/pc`
      - `xkb_geometry "pc105"`
    - `/usr/share/X11/xkb/symbols/pc`
      - `xkb_symbols "pc105"`
    - `/usr/share/X11/xkb/symbols/us`
      - `xkb_symbols "basic"`
    - `/usr/share/X11/xkb/symbols/inet`
      - `xkb_symbols "evdev"`
    - `/usr/share/X11/xkb/types/complete`
      - `default xkb_types "complete"`
  - when I click `x` on my keyboard
    - evdev reports 45 (`KEY_X`)
    - the compositor forwards 45 to the client
    - the client calls `xkb_state_key_get_syms` to translate 45 to `XKB_KEY_x`
      (0x78, same as ascii)
    - xkb?????
      - `keycodes/evdev` maps `AB02` to 53 (what xfree86 uses for `x`)
      - `geometry/pc` maps `x` to `AB02`
      - `symbols/us` maps `AB02` to `x` or `X`

## Physical Keyboard Layout

- character keys
  - to type characters and other symbols
- modifier keys
  - to modify the functions of other keys
  - `Shift`
  - `Ctrl`
  - `Alt`
  - `AltGr`
    - on US keyboard, this is just right `Alt`
  - `Cmd` and `Options` on Apple
  - `Meta`, `Hyper`, `Super`
  - dead keys
  - `Compose`
- system command keys
  - `SysRq` and `PrtScn`
  - `Break` and `Pause`
  - `Esc`
  - `Enter`
  - `Menu`
  - `Command`
  - `Window`
- levels
  - using `c` key as an example
  - first level, `c`, types `c`
  - second level, `Shift+c`, types `C`
  - third level, `AltGr+c`, types `©`
  - fourth level, `AltGr+Shift+c`, types `¢`
- pc101 / pc102 / pc104 / pc105
  - <https://en.wikipedia.org/wiki/Keyboard_layout>
  - standard us / europe layout
    - europe layout has some differences
    - the extra key is for dead key
  - pc104 / pc105 has 3 more keys
    - two `Window` and one `Menu` keys
