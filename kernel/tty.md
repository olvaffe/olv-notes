See also vt and keyboard

TTY is mainly in tty_io.c

tty_fops->open:
* struct tty_driver and index is looked up first through dev_t (major and minor device numbers)
* corresponding struct tty_struct is tty_reopen or tty_init_dev
* in tty_init_dev, tty_ldisc_setup is called, which calls tty_ldisc_ops->open and tty_ldisc_enable
* tty_operations->open is called

tty_fops->read:
* tty_ldisc_ops->read is called

tty_fops->write:
* tty_ldisc_ops->write is called
* which calls tty_operations->write

tty_fops->ioctl:
* TIOC* is handled in tty_ioctl
* tty_operations->ioctl is called.  In vt's case, KBD* and VT_*  are handled
* tty_ldisc_ops->ioctl is called at last, more TIOC* is handled
* in n_tty case, it calls n_tty_ioctl_helper, which calls tty_mode_ioctl
* tcsetattr calls TCSETSW, which is handled by tty_mode_ioctl, which calls set_termios
* then tty_operations->set_termios
* then tty_ldisc_ops->set_termios

(keyboard) Input to TTY:
* will finally be flush_to_ldisc
* which calls tty_ldisc_ops->receive_buf
* which does all kinds of stuff, echoing, crlf, sending signals, and etc.

In summary, a TTY driver:
* in init, calls alloc_tty_driver, tty_set_operations, and tty_register_driver
* tty_operations has open, close, write, ioctl, and etc.  But _no_ read.
* That is because TTY does not "read" (keyboard) input from TTY driver.
* TTY driver pushes input to TTY by tty_ldisc_ops->receive_buf

USB Serial Converter is a TTY driver:
? when urbs come in, tty_insert_flip_char or tty_flip_buffer_push is called
* which calls tty_ldisc_ops->receive_buf.
* which could be read
? when being written, urb is allocated and input is transmitted

cu:
? set raw, every keystroke is passed directly to destination
* 
