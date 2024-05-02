SSH
===

## Handshake

- client connects to server
- client and server agree on the protocol version
- client and server agree on the key exchange, host key, cipher, and mac
  algorithms
  - the key exchange algorithm allows both client and server to generate the
    same secret key over insecure channel
    - `KexAlgorithms`
  - the host key algorithm is how the host key is generated
    - `HostKeyAlgorithms`
    - the host key is how client identifies server
  - the cipher algorithm encrypts all further messages
    - `Ciphers`
  - the mac algorithm computes a hash for each further message for integrity
    - `MACs`
    - some ciphers do not need separate macs for message integrity
- client and server encrypts all messages using the agreed cipher and secrete
  key
- server authenticate client
  - `PubkeyAuthentication` uses client pubkey to authenticate client
    - server encrypts a challenge using client pubkey
    - client decrypts the challenge using client privkey
  - `PasswordAuthentication` uses client password to authenticate client
  - `HostbasedAuthentication` is similar to `PubkeyAuthentication`, except the
    key is per-host not per-client (e.g., all users on the same host can connect)
  - `KbdInteractiveAuthentication` uses bsd-speicifc `login.conf`  to
    authenticate client?
  - `GSSAPIAuthentication` uses GSSAPI to autheticate client

## Server Config

- common keywords
  - `PermitRootLogin` to restrict root login
  - `AllowUsers` to allow only listed users
  - `PasswordAuthentication` to enable/disable password auth
  - `PermitEmptyPasswords` to allow/disallow empty password

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
  - `-T`, `-t`, and `-tt` can explicit control whether a pty is allocated by
    sshd
