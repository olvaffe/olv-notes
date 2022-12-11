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
