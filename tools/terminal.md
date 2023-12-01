Terminal
========

## ncurses

- all TUI (text user interface) apps need terminal control
- in the early days,
  - there was termcap that was used by vi
  - curses initially used termcap
  - terminfo replaced termcap
- ncurses
  - free software re-implements terminfo and curses

## Life of a Key Press

- a terminal emulator allocates a pty and starts a shell whose
  stdin/stdout/stderr are connected to the pty
- when a key is pressed, the terminal emulator receives the event from the
  display server.  It feeds the key into the pty
- the pty sits between the emulator and the shell
  - it massages the data travel through it
  - for example, when Ctrl-C is pressed, the terminal feeds ascii code 0x3
    into pty; pty sends SIGINT to the foreground process (which shell sets via
    `tcsetpgrp`)
- experiment
  - open a terminal and

    $ tty
    /dev/pts/0
    $ stty -F /dev/pts/0
    speed 38400 baud; line = 0;
    -brkint -imaxbel iutf8
  - open a second terminal and

    $ stty -F /dev/pts/0
    speed 38400 baud; line = 0;
    lnext = <undef>; discard = <undef>; min = 1; time = 0;
    -brkint -icrnl -imaxbel iutf8
    -icanon -echo
  - bash uses readline for line editing
    - that puts the tty into the raw mode to handle ascii codes as keys are
      pressed
  - when bash starts a program, it restores the tty back to cooked mode
    - the tty driver handles echoing and line editing
    - the program sees ascii codes only when Enter is pressed

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
- the terminal emulator holds on to the master fd and forks a child.  The
  child opens the slave device as stdin/stdout/stderr, and execs the shell
  - writes to the master (by the terminal emulator) are received by the slave
    (by the shell)
  - writes to the slave (by the shell) are received by the master (by the
    terminal emulator)

## ANSI Escape Code

- it allows a program to send an in-band command to the terminal
- C0 control codes
  - `echo -ne`
  - `\a` sends an alert/bell
  - `\b` sends a backspace
  - `\e` sends an escape
  - `\f` sends a form feed (next page for printers)
  - `\n` sends a new line/line feed
  - `\r` sends a carriage return (to first column)
  - `\t` sends a horizontal tab
  - `\v` sends a vertical tab
- Fe escape sequences
  - `\e` followed by `\x40`..`\x5f`
    - `\x40` is `@`
    - `\x5b` is `[`
    - `\x5f` is `_`
  - `\e[` starts a CSI (control sequence introducer) sequence
    - try `echo -e '\e[31mAAA\e[0m'`
    - `\e[Xm` sets the display attribute, where `X` is a number
      - 0: reset
      - 1: bolden
      - 2: dim
      - 3: italic
      - 4: underline
      - 7: invert fg/bg colors
      - 9: strike
      - 30-37: foreground colors
        - 8 colors
        - black, red, green, yellow, blue, magenta, cyan, white
      - 38: more foreground colors
        - `38;5;Y`, where Y is from 0 to 255 for 256 colors
          - 0-7: same as X 30-37
          - 8-15: same as X 90-97
          - 16-231: 6x6x6 colors
          - 232-255: 24 grayscale
        - `38;2;R;G;B` for true colors
      - 39: default foreground color
      - 40-47: background colors
      - 48: more background color
      - 49: default background color
      - 90-97: bright foreground colors
      - 100-107: bright background colors
