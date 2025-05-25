Kernel PTY
==========

## PTY

- TTY stands for teletype terminal
- PTY stands for pseudo TTY, or pseudoterminal
- `man pty`
  - `posix_openpt` returns an fd that refers to a master
    - on linux, this `open("/dev/ptmx")`
    - previously, this opens an unused `/dev/ptyp*`
  - `grantpt` changes the owner of the slave device to the real uid of the
    caller
    - on linux, this is nop.  `/dev/pts` is a `devpts` pseudo fs.  Slave
      devices are created automatically under `devpts` when the master devices
      are opened
    - previously, this invokes an suid program to change the owner of
      `/dev/ttyp*`
  - `unlockpt` allows the slave device to be opened
    - on linux, this is `TIOCSPTLCK`
  - `ptsname` returns the path to the slave device
- the terminal emulator (or sudo or sshd) holds on to the master fd and forks a child.  The
  child opens the slave device as stdin/stdout/stderr, and execs the shell
  - writes to the master (by the terminal emulator) are received by the slave
    (by the shell)
  - writes to the slave (by the shell) are received by the master (by the
    terminal emulator)

## PTY

- `/dev/ptmx` has `tty_fops` as its `file_operations` with `open` overriden
- when `/dev/ptmx` is opened, the driver checks if `/dev/pts` is mounted as
  devpts.  If so, it gets an index from devpts and
  `tty_init_dev(ptm_driver, index)` to initialize a master device.  It also
  calls `devpts_pty_new` to create a new inode in devpts.  `/dev/pts/<n>` uses
  the standard `tty_fops`
  - `tty_init_dev` calls `ptm_driver`'s `install`.  This sets up the slave
    `tty_struct`
  - it also sets up `tty_ldisc`
- when the master is written to, it goes through
  - `tty_write`
  - `n_tty_write`
  - `pty_write` adds the data to the input queue of the slave, and triggers
    `flush_to_ldisc` on the slave
  - `n_tty_receive_buf` on the slave is called
    - Among other things, it calls `echo_char`, `commit_echoes`, and
      `flush_echoes`.  Echos are written to the slave and are received by the
      master.
    - because `ptm_driver->init_termios` misses most flags, and ioctls against
      the master are redirected to the slave, the master ldisc is always in
      `real_raw` mode.
- when the slave is read, it goes through
  - `tty_read`
  - `n_tty_read`
