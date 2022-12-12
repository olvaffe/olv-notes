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

## xkbcommon

- <https://github.com/xkbcommon/libxkbcommon/blob/master/doc/quick-guide.md>
  - the goal is to compile a `xkb_keymap`
  - we can ask the user to provide an RMLVO and

    const struct xkb_rule_names names = { RMLVO };
    xkb_keymap_new_from_names(ctx, &names, XKB_KEYMAP_COMPILE_NO_FLAGS);
  - or, if we are a wayland client, we can get the keymap from the compositor
    as a string and
    `xkb_keymap_new_from_string(ctx, string, XKB_KEYMAP_FORMAT_TEXT_V1, XKB_KEYMAP_COMPILE_NO_FLAGS)`
- a wayland client
  - `wl_keyboard::keymap` gives the keymap
  - `wl_keyboard::key` gives the keycode when a key is pressed/released
  - using xkbcommon,
    - `xkb_keymap_new_from_string` to create the keymap
    - `xkb_state_new(keymap)` to create the state
      - it remembers things like which keyboard modifiers or LEDs are active
    - `xkb_state_key_get_one_sym(state, keycode)` to translate keyboard to
      keysym
