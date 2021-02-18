Bash
====

## Invocation

- bash starts in interactive mode by default
  - `-i` makes sure it is interactive
- `login` starts bash with `argv[0][0] == '-'`
  - this makes bash a login shell
  - same as if `-l` is specified
- when bash is invoked as a login shell, interactive or not,
  - it reads `/etc/profile` first
  - it then reads the first existing file of the following files in order
    - `~/.bash_profile`
    - `~/.bash_login`
    - `~/.profile`
  - when it exits, it read `~/.bash_logout`
- when bash is invoked as a non-login interactive shell,
  - it reads `~/.bashrc`
