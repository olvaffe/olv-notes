Bash
====

## Invocation

- login shell
  - one started with `--login/-l` or `argv[0][0] == '-'`
  - `login` starts bash with `argv[0][0] == '-'`
- interactive shell
  - non-interactive: has any non-option argument or `-c`
  - otherwise, it is interactive
  - can also be forced with `-i` or `-s`
- when bash is invoked as a login shell, interactive or not,
  - it reads `/etc/profile` first
  - it then reads the first existing file of the following files in order
    - `~/.bash_profile`
    - `~/.bash_login`
    - `~/.profile`
  - when it exits, it read `~/.bash_logout`
- when bash is invoked as a non-login interactive shell,
  - it reads `/etc/bash.bashrc` and `~/.bashrc`
- when bash is invoked as `sh`
  - if a login shell, interactive or not, it reads `/etc/profile` and
    `~/.profile`
  - if a interactive non-login shell, it reads nothing

## Shell Grammar

- `!`
- `case`, `in`, `esac`
- `for`, `while`, `do`, `done`
- `if`, `then`, `elif`, `else`, `fi`
- `{` and `}`
- `time`
- `[[` and `]]`

## Prompting

- `PS1="\u@\h:\w\$ "`
  - `\u` is the user name
  - `\h` is the hostname up to the first dot
  - `\w` is the working dir
  - `\$` prints `$` or `#` depending on whether the user is root
  - `\e` prints ESC
  - `\a` prints BELL
- change terminal window title
  - `ESC]2;titleBEL` sets the window title to `title`
  - replace `2` by `0` to change the "icon name" as well
    - for WMs that understand it, "icon name" is the name to display when the
      window is iconized/minimized

## Shell Builtin Commands

- `:` is nop
- `.` or `source`
  - read and execute commands from the specifiled file using the current shell
- `alias` aliases a command
- `bg`
- `builtin` executes the specified builtin command
  - useful when a function has the same name as the builtin
- `cd`
- `command` executes the specified (builtin or external) command
  - useful when a function has the same name as the command
- `echo`
- `eval`
- `exec`
- `exit`
- `export`
- `fg`
- `help`
- `history`
- `jobs`
- `kill`
- `popd`
- `pushd`
- `read`
- `set`
- `shopt`
- `test` or `[`
- `type`
- `ulimit`
- `umask`

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
