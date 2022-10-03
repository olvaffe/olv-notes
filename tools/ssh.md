SSH
===

## Config

- for each parameter, the first obtained value from the following sources is
  used
  - cmdline options
  - `~/.ssh/config`
  - `/etc/ssh/ssh_config`
- generally, host-specific declarations are at the beginning of the config
  files and general defaults are at the end
- `Host` restricts following declarations (up to the next `Host` or `Match`)
  only to the hosts matched
  - `?` matches any one character
  - `*` matches any zero or more characters
  - `A B` matches A or B
  - `A !B` matches A unless B
- `Match` restricts following declarations (up to the next `Host` or `Match`)
  only to when the criteria are met
  - `all` is always true
  - `canonical` is true when it is after `CanonicalizeHostname`
  - `final` is true when it is the last pass
  - `exec` is true when the shell cmd exits with error code 0
  - `host` is true when the substituted host (after `Hostname` and
    `CanonicalizeHostname`) matches
  - `originalhost` is true when the original host (specified on cmdline)
    matches
  - `user` is true when the remote user name matches
  - `localuser` is true when the local user name matches

## server

- there is a sshd process running as root
  - when a connection is accepted, it forks a `[priv]` process to manage the
    connection
  - `[priv]` does not process network traffic directly; instead, it forks a
    unpriviledged `[net]` process to process authentication traffic
  - once authenticated, `[net]` exits; `[priv]` forks a user
    `some_user@some_pty` or `some_user@notty` process to handle traffic
- pty allocation
  - a pty is allocated if no command is specified
    - `ssh host` allocates a pty
    - `ssh host sh` does not
  - `-T`, `-t, and `-tt` can explicit control whether a pty is allocated by
    sshd
