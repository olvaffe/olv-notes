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
- server authenticate client - `PubkeyAuthentication` uses client pubkey to authenticate client
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
- expriment: `ssh <remote>`
  - local has a single process, `ssh <remote>` itself
    - the stdio connects to the controlling tty
    - there is a socket connecting to remote
    - the controlling tty is in raw mode
    - it forwards data between the controlling tty and the socket
  - remote has 3 processes
    - `sshd: olv [priv]` runs as root
      - it waits for the session to end and performs cleanup
      - this is similar to when a user logs in locally, `login` runs as root
        and waits for the session to end
    - `sshd: olv@pts/0` runs as user
      - the stdio connects to `/dev/null`
      - there is a socket connecting to local
      - there is a fd connecting to `/dev/ptmx`
      - it acts as a terminal emulator and forwards data between ptmx and the
        socket
    - `-bash` runs as user
      - the stdio connects to a pty (`/dev/pts/X`)
- expriment: `ssh <remote> sleep infinity`
  - local has a single process, `ssh <remote> sleep infinity` itself
    - same as above
  - remote has 3 processes
    - `sshd: olv [priv]` runs as root
      - same as above
    - `sshd: olv@notty` runs as user
      - the stdio connects to `/dev/null`
      - there is a socket connecting to local
      - there are 3 pipes connecting to `sleep infinity`
      - it forwards data between the pipes and the socket
    - `sleep infinity` runs as user
      - the stdio connects to the 3 pipes

## SSH agent

- `ssh-keygen` generates a key on disk
  - it defaults to Ed25519 and saves the privkey to `~/.ssh/id_ed25519`
  - the privkey is optionally encrypted
  - it also generates the pubkey from the privkey
    - `ssh-keygen -y` re-generates the pubkey from the privkey
- `ssh-agent` caches (decrypted) keys in memory
  - it listens on `SSH_AUTH_SOCK` for connections
- `ssh-add` manages `ssh-agent`
  - without arg, it adds decrypted `~/.ssh/id_ed25519` to the agent
  - `ssh-add -L` lists all cached keys
- `ssh` connects to `SSH_AUTH_SOCK` to get the cached keys
