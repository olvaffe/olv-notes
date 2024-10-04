Kernel TTY IO
=============

## Init

- `chr_dev_init` calls `tty_init` to register chrdevs
  - `/dev/tty` has `MKDEV(TTYAUX_MAJOR, 0)`
  - `/dev/console` has `MKDEV(TTYAUX_MAJOR, 1)`
  - if vt is enabled, `vty_init` is called
    - `/dev/tty0` has `MKDEV(TTY_MAJOR, 0)`
- `/dev/tty` uses `tty_fops`
  - on open, `tty_open` calls `tty_open_current_tty`
    - this returns `current->signal->tty`, the controlling tty
- `/dev/console` and `/dev/tty0` both use `console_fops`
  - on open, `tty_open` calls `tty_open_by_driver`
    - `tty_lookup_driver` returns the `tty_driver` and the dev idx
      - for `/dev/console`, this calls `console_device` from printk
        - it returns the first tty console in `console_list`
        - the last `console=` specifies the preferred console and
          `register_console` keeps it at the head of `console_list`
      - for `/dev/tty0`, this returns `console_driver` and `fg_console` from
        vt
