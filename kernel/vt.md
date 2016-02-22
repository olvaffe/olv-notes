Kernel VT
=========

## (old) Overview:

* See also kernel-tty and kernel-keyboard
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

* see kernel-keyboard

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
