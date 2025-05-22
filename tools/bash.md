Bash
====

## Options

- `-c`: commands are from the first non-option arg
  - `-c "a b c"` is similar to `echo "a b c" | bash`
  - `-c a b c` is similar to `echo a | bash -s b c`
- `-i`: forces interactive shell
- `-l`: forces login shell
- `-s`: assigns non-option args to positional params
  - `-s b c` assigns `b` to `$0` and `c` to `$1`

## Arguments

- if there is any non-option arg, by default,
  - the first is a script file from which commands are read and executed
  - the rest are assigned to positional params (`$1`, `$2`, etc.)
  - this is how shebang works

## Invocation

- scenarios
  - login: `-bash`
  - ssh: `-bash`
  - ssh cmd args: `bash -c "<cmd> <args>"`
  - new terminal: `/bin/bash`
  - script args: `<shebang> <script> <args>`
- a login shell is a shell started `argv[0][0] == '-'`
  - `-l` forces a login shell
- an interactive shell is a shell reading inputs from stdin (no non-option
  args) and stdin is a tty (determined by `isatty`)
  - `-i` forces an interactive shell
- when bash is invoked as an interactive login shell,
  - it reads `/etc/profile` first
  - it then reads the first existing file of the following files in order
    - `~/.bash_profile`
    - `~/.bash_login`
    - `~/.profile`
  - when it exits, it read `~/.bash_logout`
- when bash is invoked as an interactive non-login shell,
  - it reads `/etc/bash.bashrc` first
  - it then reads `~/.bashrc`
- when bash is invoked as `sh`, and as an interactive login shell,
  - it reads `/etc/profile` first
  - it then reads `~/.profile`
  - it reads nothing if invoked as an interactive non-login shell
- special cases
  - when `-l` forces a login shell, profile files are sourced even non-interactive
  - when `sshd` invokes a non-login bash, rc files are sourced even
    non-interactive
    - the idea is that `ssh remote cmd args...` is `bash -c 'cmd args...'` on
      the remote machine, but we want it to mimic `cmd args...` on the remote
      machine instead

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
