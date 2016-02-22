See also tty and vt

Linux journal:

* scancode mode: 1, 2, 3.  we discuss 2 hereafter.
* scancode is usually in the range 0x01-0x5f (+0x80 in key release)
* mode can be changed by ioctl(0, KDSKBMODE, K_RAW);
* keycode 1~127 for key press, +128 for key release
* keycode in kernel has no relation with keycode in X
* scancode == keycode if scancode == 0x01-0x58
* keymap is used for keycode->keysym
* there are 8 modifiers: Shift, AltGr, Control, Alt, ShiftL, ShiftR, CtrlL and CtrlR
* theoretically, there could be 2^8 keymaps
* keysym is two-bytes, (type, value), unless in unicode mode
* type = KTYP(keysym); (*key_handler[type])(keysym & 0xff, up_flag);
* there is a type, because certain key combinations have special effect

kernel:
* When 0x1e is mapped to 0, press a gives
  atkbd.c: Unknown key pressed (translated set 2, code 0x1e on isa0060/serio0)
* SysRq -> 0xe0 0x2a 0xe0 0x37 -> 255 99 -> ATKBD_KEY_NULL KEY_SYSRQ
* the position of 'y' provides scan code 0x15, which is mapped to KEY_Y, which could be y or z depending on keymaps

xkb:
* different driver produces different keycodes, which makes keycode meaningless
* need a corresponding XkbKeyCode for each driver to map keycode to symbolic name
* symbolic name is used in the rest of xkb to map to keysym


atkbd.c:
* perform scan code to keycode mapping (modifiable by setkeycodes and getkeycodes)
* atkbd_set_device_attrs sets device attr, which includes max scancode, etc.
* input_default_setkeycode

keyboard.c:
* k_handler, fn_handler, kbd_table for each console (kbd_struct, interfacing vt), kbd_handler (input_handler, interfacing input layer)
* if hw is capable of generating scan code, and kbd is in raw mode, scan code is put on the vt queue
* if hw is not, emulated scan code is generated
* keycode is mapped to keysym. (keymap modifiable by loadkeys, dumpkeys)
* sysrq uses kbd_sysrq_xlate instead of keymap.  For example, alt-sysrq-s works even if s is mapped to h in keymap.
* a keysym is two bytes.  if higher byte < 0xf0, it is a unicode codepoint and outputed directly
* otherwise, it indicates type and the lower byte is value.  If it is a letter, the value goes through consolemap to get a unicode codepoint.

press 'A' -> scan code of A -> keycode KEY_A -> keysym 0x61, type letter
-> conv_8bit_to_uni -> output utf8 if kbdmode == VT_UNICODE
                    -> output the result of conv_uni_to_8bit if not
