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
