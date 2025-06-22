PGP
===

## History

- PGP
  - 1991: Phil Zimmermann created PGP, Pretty Good Privacy
  - 1997: PGP Inc and PGP 5
    - there was PGP 2 from the original author
    - there was PGP 4, based on PGP 2, from another company, Viacrypt
    - there was PGP 3 developed by the original author, but renamed to PGP 5
      when the two comapanies merged
  - 1997: NAI acquired PGP Inc
  - 2002: PGP Corp bought PGP from NAI
  - 2010: Symantec acquired PGP Corp
  - 2019: Broadcom acquired a division of Symantec, including PGP
- OpenPGP
  - 1996: RFC 1991, PGP Message Exchange Formats (v3 key format, deprecated)
  - 1997: PGP Inc proposed forming OpenPGP WG in IETF
  - 1998: RFC 2440, OpenPGP Message Format (v4 key format, deprecated)
  - 2007: RFC 4880, OpenPGP Message Format (v4 key format, deprecated)
  - 2009: RFC 5581, The Camellia Cipher in OpenPGP (deprecated)
  - 2012: RFC 6637, Elliptic Curve Cryptography (ECC) in OpenPGP (deprecated)
  - 2024: RFC 9580, OpenPGP (v6 key format)
    - GnuPG author proposed RFC 4880bis draft, with v5 key format
    - OpenPGP WG went with v6 key format
    - GnuPG author created LibrePGP based on v5 key format
- GnuPG
  - 1997: GnuPG 0.0 by Werner Koch
  - 1999: GnuPG 1.0
  - 2002: GnuPG 1.2
  - 2004: GnuPG 1.4
  - 2006: GnuPG 2.0
  - 2014: GnuPG 2.1 (dev), compliant with RFC 4880
  - 2017: GnuPG 2.2
  - 2021: GnuPG 2.3 (dev)
  - 2022: GnuPG 2.4
  - 2024: GnuPG 2.5 (dev), compliant with LibrePGP instead of RFC 9580

## Guides

- <https://github.com/lfit/itpol/blob/master/protecting-code-integrity.md>
  - <https://www.kernel.org/doc/html/latest/process/maintainer-pgp-guide.html>
- a GPG key is usually a collection of subkeys
  - all subkeys are fully independent from each other
    - if the private key of a subkey is lost, it is gone forever
  - each subkey can have one or more capabilities
  - only one subkey can have `C` capability
    - this is often called the master key
- there are 4 capabilities
  - `S` for signing
  - `E` for encryption
  - `A` for authentication
  - `C` for certifying other keys
    - it can add/revoke other subkeys
    - it can add/change/revoke associated uids
    - it can add/change expiration on self or other subkeys
    - it can sign other people's keys
- by default,
  - when gpg generates a key,
    - it generates a subkey with `SC`
    - it generates another subkey with `E`
  - gpg encrypts private subkeys with a passphrase
    - this is essential as the last line of defense
- Generating and protecting your certification key
  - `gpg --quick-generate-key 'Name <name@example.net>' ed25519 cert`
    - this generates a PGP key for the specified uid
    - the algorithm is `ed25519`
    - it contains only a `C` subkey
    - it is passphrase-protected
    - this is a digital identity that cannot afford loss/theft
      - to avoid loss, it must be backed up
      - to avoid theft, it must be stored offline
  - `gpg --export-secret-key <fingerprint>` exports the private key for backup
    - e.g., pipe it through `paperkey`, print to a paper, and store the paper
      in back vault to avoid loss
    - we still remove the `C` subkey later to avoid theft
  - `gpg --quick-add-uid <fingerprint> 'Another <another@example.net>'` adds
    other uids associated wit this digital identity
    - it is common to have multiple uids (emails) associated with the same
      identity
    - it is also not uncommon for one to have multitiple digital identities;
      in that case, create multiple PGP keys
  - `gpg --quick-set-primary-uid <fingerprint> 'Name <name@example.net>'`
    makes an uid primary (first)
    - otherwise, each newly added uid becomes the primary (first)
  - `gpg --list-key` confirms everything looks right
- Generating PGP subkeys
  - `gpg --quick-add-key <fingerprint> ed25519 sign` adds a signing subkey
  - `gpg --quick-add-key <fingerprint> cv25519 encr` adds an encryption
    subkey
  - `gpg --quick-add-key <fingerprint> ed25519 auth` adds a authentication
    subkey (for ssh)
  - `gpg --export <fingerprint> | curl -T - https://keys.openpgp.org`
    uploads the PGP key
- Moving your certification key to offline storage
  - prepare two encrypted usb sticks
    - it is fine to use the same passphrase
  - `cp -rp ~/.gnupg /mnt/gnupg` backs up to encrypted usb sticks
  - `gpg --homedir=/mnt/gnupg-backup --list-key` verifies backups
  - remove usb sticks and put them to a safe yet convenient place
    - we will remove `C` subkey from the device to avoid theft, so we will
      need the usb sticks every now and then
  - `gpg --list-key --with-keygrip <fingerprint>` lists keys with their
    "filenames"
  - `rm ~/.gnupg/private-keys-v1.d/<keygrid>.key` removes the `C` private
    key
    - it is the digital identity so it cannot afford theft
  - `gpg --list-secret-keys` lists private keys
    - the line for the `C` subkey should go from `sec` to `sec#`, indicating
      that it is missing
  - `rm ~/.gnupg/openpgp-revocs.d/<fingerprint>.rev` removes the revocation
    certificate from the device
    - it can be used to revoke the PGP key so it cannot afford theft

## Steps

- rationale
  - a pgp key is a digital identity
    - it is common for a digital identity to have multiple emails (uids)
    - if one has multiple digital identities, one should create multiple pgp
      keys
  - `C` subkey represents the digital identity
    - it signs subkeys, uids, expiracy, revocations, etc.
    - it builds web of trust, and thus cannot afford loss/theft
    - to avoid loss, it should be backed up
    - to avoid theft, it should only be stored offline
      - this is not too inconvenient because `C` subkey is only used
        occasionally
  - `S` subkey signs messages
    - loss is fine
    - theft results in impersonation
  - `E` subkey encrypts/decrypts messages
    - loss results in losing access to encrypted messages forever
    - theft results in leaks
  - `A` subkey authenticates the digital identity
    - loss results in losing access to remote systems temporarily
    - theft results in hacks
  - per-identity or per-machine
    - there is only one `C` subkey, stored offline
    - there should be one `S` subkey, shared by all machines
      - otherwise people get confused
    - there should be one `E` subkey, shared by all machines
      - otherwise people get confused
    - there should be one `A` subkey, shared by all machines
      - otherwise I get confused
      - this may or may not be a good idea
- create a new key
  - `pkill gpg-agent; mkdir -m 700 new`
  - `gpg --homedir=new --quick-generate-key "name <email>" ed25519 cert`
  - `gpg --homedir=new --export-secret-keys --output gpg-export-secret-keys.gpg`
  - `rm -rf new`
- add subkeys to the key
  - `pkill gpg-agent; mkdir -m 700 work`
  - `gpg --homedir=work --import gpg-export-secret-keys.gpg`
  - `gpg --homedir=work --quick-set-ownertrust <fpr> ultimate`
  - `gpg --homedir=work --quick-add-key <fpr> ed25519 sign`
  - `gpg --homedir=work --quick-add-key <fpr> cv25519 encr`
  - `gpg --homedir=work --quick-add-key <fpr> ed25519 auth`
  - `gpg --homedir=work --export-secret-keys --output gpg-export-secret-keys.gpg`
  - `gpg --homedir=work --export-secret-subkeys --output gpg-export-secret-subkeys.gpg`
  - `rm -rf work`
- import subkeys
  - `gpg --import gpg-export-secret-subkeys.gpg`
  - `gpg --quick-set-ownertrust <fpr> ultimate`
  - `gpg-connect-agent "KEYATTR <keygrip> Use-for-ssh: true" /bye`

## SSH

- `gpg --export-ssh-key <fpr>` exports the public key for `authorized_keys`
- `gpg-connect-agent "KEYATTR <keygrip> Use-for-ssh: true" /bye` allows
  `gpg-agent` to cache the specified private key
  - `gpg-agent` only caches keys with `Use-for-ssh` attr or listed in
    `sshcontrol`
  - `ssh-add -L` lists cached keys
  - `ssh-add` adds an external private key
    - this adds the external private key to `private-keys-v1.d` and updates
      `sshcontrol`
- `gpg-connect-agent UPDATESTARTUPTTY /bye` updates startup env of `gpg-agent`
  - when `gpg-agent` starts, it inits `opt.startup_env` from its env
  - most tools connect to `gpg-agent` using assuan protocol
    - each connection has `ctrl->session_env` from the tool's env
    - `UPDATESTARTUPTTY` copies `ctrl->session_env` to `opt.startup_env`
  - `ssh` however connects to `gpg-agent` using `ssh-agent` protocol
    - the connection has `ctrl->session_env` copied from `opt.startup_env`
  - when any tool requests a pin, `gpg-agent` invokes `pinentry` using
    `ctrl->session_env`

## Tools

- `gpgconf`
  - `--list-components` (default) calls `gc_component_list_components`
    - it lists components in the global `gc_component` array
    - `gnupg_module_name` returns the executable path
      - e.g., `GNUPG_MODULE_NAME_AGENT` is `gpupg_bindir() + "gpg-agent"`
  - `--list-dirs` calls `list_dirs`
    - it lists dirs on `list` array
    - `sysconfdir` calls `gnupg_sysconfdir` which returns `GNUPG_SYSCONFDIR`
    - `bindir` calls `gnupg_bindir` which returns `GNUPG_BINDIR`
    - `socketdir` calls `gnupg_socketdir` which returns `/run/user/<uid>/gnupg`
    - `agent-ssh-socket` is `gnupg_socketdir` plus `GPG_AGENT_SSH_SOCK_NAME`
    - more
- `gpg-connect-agent`
  - `start_agent` starts `gpg-agent` on demand
    - `session_env_new` creates an empty session env
    - `start_new_gpg_agent` starts `gpg-agent`
      - `send_pinentry_environment` sends pinentry env
        - `GPG_TTY` becomes `OPTION ttyname=<GPG_TTY>`
        - `TERM` becomes `OPTION ttytype=<TERM>`
        - `DISPLAY` becomes `OPTION display=<DISPLAY>`
        - `WAYLAND_DISPLAY` becomes `OPTION putenv:WAYLAND_DISPLAY=<WAYLAND_DISPLAY>`
        - more
  - control cmds start with `/` and are handled by `gpg-connect-agent`
    - `/help` to list all control cmds
  - other cmds are sent to `gpg-agent`
    - `help` to list all cmds
    - `register_commands` registers cmds
    - `UPDATESTARTUPTTY` is handled by `cmd_updatestartuptty`
      - each `gpg-connect-agent` (or other) invocation calls
        `send_pinentry_environment` to update `ctrl->session_env`
      - this cmd copies `ctrl->session_env` to `opt.startup_env`
    - `GETINFO` is handled by `cmd_getinfo`
      - `std_session_env` dumps `ctrl->session_env`
      - `std_startup_env` dumps `opt.startup_env`
- `pinentry`
  - `gpg-agent` calls `start_pinentry` to invoke `pinentry`
    - if `ctrl->session_env` has `DISPLAY`, it is passed as `--display <DISPLAY>`
