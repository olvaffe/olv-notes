Terminal
========

## DEC Terminals and Emulators

- VT100 series, 1978
  - one of the first terminals to support ANSI escape code
  - xterm is closest to VT102 until 1996
  - `infocmp vt100 vt102`
    - delete character
    - delete line
    - insert line
    - exit insert mode
    - enter insert mode
- VT200 series, 1983
  - xterm is closest to VT220 until 2012
  - `infocmp vt102 vt220`
    - make cursor invisible
    - make cursor appear normal
    - visible bell
    - next-page key
    - previous-page key
    - insert-character key
    - delete-character key
- VT300 series, 1987
  - `infocmp vt220 vt320`
    - home key
  - VT340 is the first one with colors
- VT420, 1990
  - xterm is closest to VT420 since 2012
- `TERM`
  - kernel sets `TERM` to `linux` for pid 1
  - systemd sets `TERM` to
    - `linux` on `/dev/ttyX`
    - `vt220` otherwise, like on serial consoles
  - terminal emulators sets `TERM` to their owns
  - ssh forwards `TERM`
- size
  - `TIOCSWINSZ` and `TIOCGWINSZ` sets/gets the window size
    - `SIGWINCH` notifies size change
  - apps get window size and monitor size change
  - kernel vt inits `tty->winsize` in `con_install`
    - kernel serial has no default size
  - terminal emulators set terminal size
  - ssh forward terminal size
  - agetty sets terminal size to 80x24 if it is currently 0x0
  - systemd performs a tty reset when `TTYReset=yes`
    - this is the case for `getty@.service` and `serial-getty@.service`
    - `terminal_get_size_by_dsr` sends ansi escope codes to query the terminal
      size
      - query current cursor pos
      - set cursor pos to high val
      - query cursor pos, which is capped to terminal size
      - restore cursor pos
    - `terminal_set_size_fd` sets the terminal size
- serial console
  - remote programs communicate over the physical serial port on the remote
    machine
    - `TERM` and `stty size` often have incorrect values on the remote machine
    - `stty cols <val1> rows <val2>` to correct
  - we also need a terminal emulator that communicates over a local physical
    serial port rather than over pty on the local machine
    - picocom
      - `picocom -b <baud> <dev>` to connect
      - `C-a C-x` to disconnect
      - it forwards traffic between the serial port and the host terminal
    - cu
      - `cu -s <baud> -l <dev> -f` to connect
      - `~.` to disconnect
      - it forwards traffic between the serial port and the host terminal
    - minicom
      - `minicom -b <baud> -D <dev>` to connect
      - `C-a x` to disconnect
      - it emulates a vt102 terminal on top of the host terminal
        - `-c on` to enable color support

## ncurses

- all TUI (text user interface) apps need terminal control
- in the early days,
  - there was termcap that was used by vi
  - curses initially used termcap
  - terminfo replaced termcap
- ncurses
  - free software re-implements terminfo and curses

## Character Sets

- ASCII
  - <https://en.wikipedia.org/wiki/ASCII>
  - 7-bit
  - developed by ANSI in 1960s
    - founded in 1918 as AESC (American Engineering Standards Committee)
    - reorganized in 1928 to ASA (American Standards Association)
    - reorganized in 1966 to USASI (United States of America Standards Institute)
    - reorganized in 1969 to ANSI (American National Standards Institute)
  - every 16 positions is called a stick, e.g., `0x30..0x3f` is stick 3
  - character groups
    - 0x00..0x1f: 32 control codes (stick 0 and 1)
      - some remain but most are unused
      - `\r`, `\t`, `\n`, backspace, escape, etc.
      - usually take control+key combinations to generate on keyboards
    - 0x20..0x7e: 95 printable characters
      - stick 2: symbols
      - stick 3: digits (0 is 0x30) and more symbols
      - stick 4&5: uppercase letters (A is 0x41) and more symbols
      - stick 6&7: lowercase letters (a is 0x61) and more symbols
    - 0x7f: DEL control code
- ISO/IEC 646
  - <https://en.wikipedia.org/wiki/ISO/IEC_646>
  - 7-bit
  - ISO 646:1991 IRV (International Reference Version) is identical to ASCII
  - there are national variants where symbols can vary
    - in JP variant, `\` is replaced by `¥`
    - in GB variant, `#` is replaced by `£`
- ISO/IEC 8859
  - <https://en.wikipedia.org/wiki/ISO/IEC_8859>
  - 8-bit
  - there are 16 parts (variants)
  - ISO/IEC 8859-1
    - this is part 1 and is the most common one
    - 0x00..0x1f: 32 undefined codes
    - 0x20..0x7e: 95 printable characters
      - identical to ASCII
    - 0x7f..0x9f: 33 undefined codes
    - 0xa0..0xff: 96 printable characters
      - symbols and other latin letters
- ISO/IEC 10646
  - <https://en.wikipedia.org/wiki/Universal_Coded_Character_Set>
  - similar to ISO 8859, it only defines the code points
    - the code points are the same as unicode
    - unicode defines rules (normalization, composition, collation,
      directionality) addtionally
  - UCS, Universal Coded Character Set, defines 2^31 code points
    - 7-bit group
    - 8-bit plane
    - 8-bit row
    - 8-bit cell
  - BMP, Basic Multilingual Plane, has group 0 and plane 0
  - ISO 8859-1 has group 0, plane 0, and row 0
  - UTF-32 (UCS-4) encoding
    - each code point takes 4 bytes
    - big endian with group at the top and cell at the bottom
  - UTF-16 (UCS-2) encoding
    - each code point takes 2 bytes
    - there is a BOM character to indicate the endianness
    - direct mapping for code points in BMP
    - surrogate characters are needed for other planes
  - UTF-8 encoding
    - variable length
      - 1 byte can encode 7 bits
      - N bytes can encode `(8 - N - 1) + 6 * (N - 1)` bits, when N > 1
    - 1 byte to cover ASCII
    - 2 bytes to cover ISO 8859-1
    - 3 bytes to cover BMP
    - 6 bytes to cover UCS

## ANSI Escape Code

- ISO/IEC 6429
  - <https://en.wikipedia.org/wiki/ANSI_escape_code>
  - ISO 6429 defines C0 (0x00..0x1f) and C1 (0x7f..0x9f) control codes using
    the undefined regions of ISO 8859-1
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
