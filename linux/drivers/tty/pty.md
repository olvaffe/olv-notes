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
