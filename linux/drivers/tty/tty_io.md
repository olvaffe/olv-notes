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

## `tty_driver`

- other subsystems may call `tty_register_driver` to register tty drivers
  - bt rfcomm registers `rfcomm_tty_driver` for `/dev/rfcomm*`
  - usb serial registers `usb_serial_tty_driver` for `/dev/ttyUSB*`
  - usb cdc-acm registers `acm_tty_driver` for `/dev/ttyACM*`
  - usb cdc-acm registers `acm_tty_driver` for `/dev/ttyACM*`
  - sdio uart registers `tty_drv` for `/dev/ttySDIO*`
- tty subsystem has these `tty_driver`
  - pty registers `ptm_driver` and `pts_driver`
    - legacy pty registers `pty_driver` and `pty_slave_driver`
  - rpmsg registers `rpmsg_tty_driver` for `/dev/ttyRPMSG*`
  - serial registers `normal`
    - `/dev/ttyS*` for 8250 and several others
    - `/dev/ttyAMA*` for AMBA PL011
    - `/dev/ttyMSM*` for qcom MSM and GENI
    - `/dev/ttyHS*` for qcom GENI
  - ttynull registers for `/dev/ttynull`
  - vt registers `console_driver` for `/dev/tty*`
- each driver has a `tty_operations`
  - tty fops forwards many ops to `tty_operations`

## `file_operations`

- the tty subsystem has these `file_operations`
  - `tty_fops`
  - `console_fops` is only used by `/dev/console` and `/dev/tty0`
    - it is the same as `tty_fops` except it supports `TIOCCONS` redirect
  - `hung_up_tty_fops` is only used after a tty is hung up
    - it stubs out all ops
- `tty_fops::open` is `tty_open`
  - if rdev is `MKDEV(TTYAUX_MAJOR, 0)`, that is `/dev/tty`,
    `tty_open_current_tty` returns the controlling tty
  - otherwise, `tty_open_by_driver` opens the tty
    - if rdev is `MKDEV(TTY_MAJOR, 0)`, that is `/dev/tty0`, it uses vt
      `console_driver`
    - if rdev is `MKDEV(TTYAUX_MAJOR, 1)`, that is `/dev/console`, it uses
      printk `console_device`
    - otherwise, `get_tty_driver` searches `tty_drivers` for drivers
      registered with `tty_register_driver`
  - `tty_init_dev` calls `tty_ldisc_init` indirectly and uses `N_TTY` line
    discipline by default
- `tty_fops::read_iter` is `tty_read` and reads via ldisc
  - tty drivers typically call `tty_flip_buffer_push` to push data to ldisc
    for reading
- `tty_fops::write_iter` is `tty_write` and writes via ldisc
- `tty_fops::unlocked_ioctl` is `tty_ioctl`
  - it handles a bunch of `TIOC*` ioctls
  - `tty_jobctrl_ioctl` handles more `TIOC*` ioctls related to job control
  - the tty driver handles more ioctls
    - serial handles some `TIOC*` ioctls
    - vt handles `KD*` and `VT_*` ioctls
  - the ldisc handles `TIOC*` and `TC*` ioctls
