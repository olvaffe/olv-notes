Kernel VT
=========

## Overview

- VT is a terminal emulator
- `vty_init` calls `tty_register_driver` to provide tty lines to userspace
  - `vcs_init` provides `/dev/vcsX` for screendump
  - when userspace opens a tty line, `tty_init_dev` creates a `tty_struct`
    and `con_install` inits a `vc_data` on `vc_cons` array
    - each `vc_data` has a `tty_port`
- `kbd_init` calls `input_register_handler` to get input events from the
  input subsys
- fbdev calls `do_take_over_console` to provide `consw` for output

## `struct consw` and `struct console`

- `struct console` is used with `register_console` to register a console to
  printk
  - it is implemented by serial drivers, vt, etc. to show printk messages
- `struct consw` is used with `do_take_over_console` to register a drawing
  driver to vt
  - it is implemented by vgacon, dummycon, fbcon, etc.
  - vt provides `MAX_NR_CONSOLES` (63) virtual terminals that are internally
    called virtual consoles.  Virtual consoles need `struct consw` to draw.
- each virtual console of VT can be driven by fbcon or vgacon (or others)
  - fbcon is a midlayer and requires fbdevs to work
  - a fbdev is registered with `register_framebuffer`.  Examples are drmfb,
    vesafb, efifb, simplefb, etc.
    - drmfb is again a midlayer between fbdev and drm

## Boot

- printk's `console_init` is called very early
  - it calls vt's `con_init` to use dummycon's `dummy_con` as the consw
- `fbmem_init` is called during subsys init.  But there is no fbdev yet.
- `sysfb_init` adds a "efi-framebuffer" platform device
- `efifb_probe` calls `register_framebuffer` to registers the fbdev
   - fbcon takes over the vt consoles from dummycon
- `drm_fb_helper_initial_config` called by i915 to register "inteldrmfb"
  - `do_remove_conflicting_framebuffers` unregisters efifb first
  - inteldrmfb takes over

## Input

- when an input event occurs, `kbd_event` calls `kbd_keycode`
  - `keysym = key_map[keycode]` maps `keycode` to `keysym`
  - `type` is `KT_LATIN` (`KT_LETTER` is also forced to `KT_LATIN`)
  - `k_handler[type]` is `k_self`
    - `conv_8bit_to_uni` converts keysym to unicode
    - `conv_uni_to_8bit` converts unicode back to keysym
    - `put_queue` adds the keysym to `vc->port`
      - `tty_insert_flip_char` adds a char to `port->buf`
      - `tty_flip_buffer_push` queues `flush_to_ldisc`
- `flush_to_ldisc` pushes `port->buf` to `tty->disc_data`
  - `flush_to_ldisc` calls `tty_port_default_receive_buf`
  - `n_tty_receive_buf` calls `n_tty_receive_char` to add the data to
    `tty->disc_data`
- when userspace reads from the tty line, `tty_read` calls `n_tty_read`
  - `copy_from_read_buf` copies data out of `tty->disc_data`

## Ctrl-C and others

- ctrl has keycode `KEY_LEFTCTRL` (29)
  - `ctrl_map` maps it to keysym `0xf702`, which is handled by `k_shift`
- c has keycode `KEY_C` (46)
  - `ctrl_map` maps it to keysym `0xf003`, which is handled by `k_self`
- this causes ascii `0x3` to be inserted with `tty_insert_flip_char`
- on the receiving end, `n_tty_receive_signal_char` is called to send `SIGINT`
- other key combinations are handled similarly
  - Ctrl-S / Ctrl-Q sends 0x13 / 0x11 to stop / start tty

## Input from keyboard:

- keyboard.c:
  - k_handler, fn_handler, kbd_table for each console (kbd_struct, interfacing vt), kbd_handler (input_handler, interfacing input layer)
  - if hw is capable of generating scan code, and kbd is in raw mode, scan code is put on the vt queue
  - if hw is not, emulated scan code is generated
  - keycode is mapped to keysym. (keymap modifiable by loadkeys, dumpkeys)
  - sysrq uses kbd_sysrq_xlate instead of keymap.  For example, alt-sysrq-s works even if s is mapped to h in keymap.
  - a keysym is two bytes.  if higher byte < 0xf0, it is a unicode codepoint and outputed directly
  - otherwise, it indicates type and the lower byte is value.  If it is a letter, the value goes through consolemap to get a unicode codepoint.
- scancode mode: 1, 2, 3.  we discuss 2 hereafter.
- scancode is usually in the range 0x01-0x5f (+0x80 in key release)
- mode can be changed by ioctl(0, KDSKBMODE, K_RAW);
- keycode 1~127 for key press, +128 for key release
- keycode in kernel has no relation with keycode in X
- scancode == keycode if scancode == 0x01-0x58
- keymap is used for keycode->keysym
- there are 8 modifiers: Shift, AltGr, Control, Alt, ShiftL, ShiftR, CtrlL and CtrlR
- theoretically, there could be 2^8 keymaps
- keysym is two-bytes, (type, value), unless in unicode mode
- type = KTYP(keysym); (*key_handler[type])(keysym & 0xff, up_flag);
- there is a type, because certain key combinations have special effect

kernel:
- When 0x1e is mapped to 0, press a gives
  atkbd.c: Unknown key pressed (translated set 2, code 0x1e on isa0060/serio0)
- SysRq -> 0xe0 0x2a 0xe0 0x37 -> 255 99 -> ATKBD_KEY_NULL KEY_SYSRQ
- the position of 'y' provides scan code 0x15, which is mapped to KEY_Y, which could be y or z depending on keymaps

xkb:
- different driver produces different keycodes, which makes keycode meaningless
- need a corresponding XkbKeyCode for each driver to map keycode to symbolic name
- symbolic name is used in the rest of xkb to map to keysym


atkbd.c:
- perform scan code to keycode mapping (modifiable by setkeycodes and getkeycodes)
- atkbd_set_device_attrs sets device attr, which includes max scancode, etc.
- input_default_setkeycode

press 'A' -> scan code of A -> keycode KEY_A -> keysym 0x61, type letter
-> conv_8bit_to_uni -> output utf8 if kbdmode == VT_UNICODE
                    -> output the result of conv_uni_to_8bit if not

## Output to VT:

- console_codes(4)
- fonts have 256 or 512 glyphs, indicated by vc->vc_hi_font_mask
- in utf8 mode, input is converted to ucs; otherwise, vc_translate is used to map to ucs
- conv_uni_to_pc is called to map ucs to pc and pc is written to the hw driver.
- in summary, In utf8, input to vc are assumed to be utf8, which is converted to ucs2.
- ucs2 is converted to glyph index and sent to video memory
- In non-utf8, user can choose how to translate input (input -> ucs2 mapping)

## consolemap.c

- uni_pagedir maps unicode codepoint to fontpos (glyph index)
- see 759448f459234bfcf34b82471f0dba77a9aca498
- see console_codes(4)
- to save space, UCS2 is splitted into bits 15-11, 10-6, 5-0 to index into pgdir
- con_free_unimap unset vc->vc_uni_pagedir_loc and free it
- con_unify_unimap checks if p is the same as some other console's uni_pagedir, if so conp uses that one and free p
- con_insert_unipair manipulates the contents of uni_pagedir
- con_clear_unimap clears vc's unimap.  if it is shared, a new empty one is allocated.
- con_set_unimap sets the unimap.  if it is shared, it is copied first.  inverse translations are updated.
- set_inverse_transl builds glyph index -> (unicode ->) 8bit mapping
- set_inverse_trans_unicode builds glyph index -> unicode mapping
- conv_uni_to_pc converts ucs to glyph index
- con_set_default_unimap uses the default unimap, consolemap_deftbl.c.

## Encodings

- Kernel
  - hardware drivers handle interrupts and send input events to the input
    layer, including `EV_MSC/MSC_RAW`, `EV_MSC/MSC_SCAN`, `EV_KEY/keycode`,
    etc.
    - `MSC_RAW` is the raw data generated by the hardware
    - `MSC_SCAN` is the scancode
    - `EV_KEY` is the keycode
    - to see the events, `evtest <dev>`
    - use `EVIOCSKEYCODE_V2` to change the scancode-to-keycode mapping
  - terminal emulators, such as vt, receives input events from the input layer
    - to see the scancodes or keycodes, use `showkey -s` or `showkey -k`
    - use `KDSETKEYCODE`, or `setkeycodes`, to change the scancode-to-keycode
      mapping (same effect as `EVIOCSKEYCODE_V2`)
    - keycode sequences are further mapped to keysyms by the terminal
      emulators
    - use `KDSKBENT`, or `loadkeys`, to change the keycode-to-keysym mapping
