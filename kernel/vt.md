Kernel VT
=========

## Ctrl-C and others

- ctrl has keycode `KEY_LEFTCTRL` (29)
  - `ctrl_map` maps it to keysym `0xf702`, which is handled by `k_shift`
- c has keycode `KEY_C` (46)
  - `ctrl_map` maps it to keysym `0xf003`, which is handled by `k_self`
- this causes ascii `0x3` to be inserted with `tty_insert_flip_char`
- on the receiving end, `n_tty_receive_signal_char` is called to send `SIGINT`
- other key combinations are handled similarly
  - Ctrl-S / Ctrl-Q sends 0x13 / 0x11 to stop / start tty

## for display, I guess

* there is a system-wide console driver for all VTs
* a VT draws its content using its console driver
* we can let another console driver take over VT
  * fbcon replacing the system-wide vgacon
  * more features such as hi-res, fonts, penguin logo, etc.
* we can also ask VT not to draw its content
  * userspace draws to the display

## (old) Overview:

* See also tty.md
* /dev/tty == tty of current process
* /dev/console == first console that is also a tty
* /dev/tty0 == foreground virtual console of vt
* video/console/vgacon.c: provide struct consw
* serial/8250.c: provide struct console
* char/vt.c: (optionally) provide struct console
* consoles are registered to printk, they are `_consoles_`.
* consws (drivers) are registered to vt, they are used to implement vt console.
* consw specifies the the VTs it wants to support.  could be changed by take_over_console.
* conswitchp is the default driver. (set in the arch init phase)
* virtual terminals (struct vc) provide by vt are accessible through vc_cons.

  console_init -> console_initcalls -> (vt) con_init ->

* init has /dev/console as stdio/stdout/stderr:
  open /dev/console -> first console supporting tty_driver is used -> in case of vt, usually it means fgconsole

## data structures

* struct console: printk console (does not need tty)
* struct consw: vt driver
* struct vc: VT console (vc_cons)
* struct tty_struct: /dev/ttyX device
* struct tty_driver: driver of tty device (global console_driver of vt.c)

## Boot time

* conswitchp points to vga_con, the default driver of VT.
* console_init is called early in init/main.c.  N_TTY is registered and console_initcall are called.
* consoles, especially VT is (partly) initialized
* VT is a console, which allows multiple virtual consoles.  There are MAX_NR_CONSOLES virtual consoles (vc_cons), served by MAX_NR_CON_DRIVER drivers.
* con_driver_map maps virtual console number to driver
* 1 (the first) virtual console is initialized by visual_init and vc_init.  This virtual console is used as early printk console and does not need tty.
* register_console is called to register VT as printk console (struct console)

## Initcalls

* tty_io.c has module_init(tty_init), so tty_init is called in initcalls
* special devices /dev/tty and /dev/console are created
* vt.c:vty_init is called to really init VT
* /dev/tty0 is created
* vc_screen.c:vcs_init called to provide /dev/vcsX for screendump
* global console_driver initialized by alloc_tty_driver.
* console_driver has termios tty_std_termios and operations (vt.c's) con_ops, and then register with tty by tty_register_driver.
* after register, /dev/tty[1-63] is created, with op (tty_io.c) tty_fops
* kbd_init and console_map_init

## Userspace

? getty opens /dev/ttyX, which is represented by struct tty_struct
* when opened, if first time, tty_init_dev is called to allocate struct tty_struct
* which calls vt.c's con_ops->open, and then vc_cons[X] is allocated by vc_allocate
* vc_allocate calls visual_init, con_set_default_unimap, allocate vc_screenbuf, and vc_init.
* then tty->driver_data points to vc, and vc->vc_tty points to tty.

## Input from keyboard:

* keyboard.c:
  * k_handler, fn_handler, kbd_table for each console (kbd_struct, interfacing vt), kbd_handler (input_handler, interfacing input layer)
  * if hw is capable of generating scan code, and kbd is in raw mode, scan code is put on the vt queue
  * if hw is not, emulated scan code is generated
  * keycode is mapped to keysym. (keymap modifiable by loadkeys, dumpkeys)
  * sysrq uses kbd_sysrq_xlate instead of keymap.  For example, alt-sysrq-s works even if s is mapped to h in keymap.
  * a keysym is two bytes.  if higher byte < 0xf0, it is a unicode codepoint and outputed directly
  * otherwise, it indicates type and the lower byte is value.  If it is a letter, the value goes through consolemap to get a unicode codepoint.
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

press 'A' -> scan code of A -> keycode KEY_A -> keysym 0x61, type letter
-> conv_8bit_to_uni -> output utf8 if kbdmode == VT_UNICODE
                    -> output the result of conv_uni_to_8bit if not

## Output to VT:

* console_codes(4)
* fonts have 256 or 512 glyphs, indicated by vc->vc_hi_font_mask
* in utf8 mode, input is converted to ucs; otherwise, vc_translate is used to map to ucs
* conv_uni_to_pc is called to map ucs to pc and pc is written to the hw driver.
* in summary, In utf8, input to vc are assumed to be utf8, which is converted to ucs2.
* ucs2 is converted to glyph index and sent to video memory
* In non-utf8, user can choose how to translate input (input -> ucs2 mapping)

## consolemap.c

* uni_pagedir maps unicode codepoint to fontpos (glyph index)
* see 759448f459234bfcf34b82471f0dba77a9aca498
* see console_codes(4)
* to save space, UCS2 is splitted into bits 15-11, 10-6, 5-0 to index into pgdir
* con_free_unimap unset vc->vc_uni_pagedir_loc and free it
* con_unify_unimap checks if p is the same as some other console's uni_pagedir, if so conp uses that one and free p
* con_insert_unipair manipulates the contents of uni_pagedir
* con_clear_unimap clears vc's unimap.  if it is shared, a new empty one is allocated.
* con_set_unimap sets the unimap.  if it is shared, it is copied first.  inverse translations are updated.
* set_inverse_transl builds glyph index -> (unicode ->) 8bit mapping
* set_inverse_trans_unicode builds glyph index -> unicode mapping
* conv_uni_to_pc converts ucs to glyph index
* con_set_default_unimap uses the default unimap, consolemap_deftbl.c.

## SUMMARY:

* TTY is sitting in the center
  * it takes input from keyboard
  * it outputs to vt
  * it provides /dev/ttyX for userspace interfacing
  * it is one kind of line discipline (N_TTY)
* meanwhile, vt is also a printk console
* printk console has no keyboard input, does not need line discipline
* thus vt is partly initialized early



Screen   -> Video Driver  \
                           > VT -> Line Discipline -> TTY -> userspace
Input    -> Input Driver  /



userspace=cu -l /dev/ttyACM0 -> TTY -> Line Discipline -> Serial Core -> UART driver -> UART
		 -> UART driver -> Serial Core -> Line Discipline -> TTY -> userspace

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
- ASCII, American Standard Code for Information Interchange, character encoding
  - 7-bit encoding
  - every 16 positions is called a stick
  - stick 0 and stick 1 are control characters
    - \r, \t, \n, backspace, escape, etc.
    - usually take control+key combinations to generate on keyboards
  - 0x7f is Delete, also a control character
  - stick 2 to stick 7 are printable characters
    - stick 2 are symbols, such as space, dollar, plus, etc.
    - stick 3 are numbers and more symbols
    - stick 4 and 5 are uppercases and more symbols
    - stick 6 and 7 are lowercases and more symbols
  - terminals look for "escape sequences" and interpret them as commands
    - `ESC + [` is the Control Sequence Introducer 
- terminfo
  - describe the capabilities of a terminal emulator
  - `infocmp` to decompile a terminfo

## fbdev

- In kernel, a fbdev is a VT driver.  That is how VT draws characters on screen.
- The first thing to do is to switch to a new VT and ask the VT not to draw to
  the screen
  - `ioctl(fd, VT_ACTIVATE, vtno)`
  - `ioctl(fd, KDSETMODE, KD_GRAPHICS)`
- Then we can open the fbdev device and draw at will
- Xorg
  - observed via `ps -ax -o sid,comm,tpgid,pid,tty` and the source
  - The server is in its own session.
  - It has the new VT as the controlling terminal
    - to set the new VT to graphics mode
    - to avoid other processes stealing the VT
  - It is also the foreground process of the new VT
  - Input events are received from evdev
    - the new VT is constantly flushed to keep its buffer clean
    - the new VT is made raw to avoid signals and others
  - It is also the foreground process of the old VT
    - it can receive Ctrl-C from the old VT
