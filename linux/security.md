Kernel Security
===============

## SELinux

- Mandatory Access Controls (MACs)
  - separate from other access controls such as filesystem permissions or sudo
  - can restrict a root
- `CONFIG_SECURITY_SELINUX` is default off
  - distros usually have it on
  - but userspace tools might not have `libselinux` support
  - and selinux policies might be missing
- to enable selinux in kernel, pass `security=selinux selinux=1`
- `pam_selinux` sets up security context for the session
- reference policy
  - `make bare`
  - `make conf`
  - `make policy`
  - `make install`
  - `make load`
- `/etc/selinux/config`
  - `SELINUX=permissive`
  - `SELINUXTYPE=refpolicy`
- userspace packages
  - `checkpolicy`
    - `checkpolicy`
    - `checkmodule`
  - `policycoreutils`
    - `fixfiles`
    - `load_policy`
    - `restorecon`
    - `restorecon_xattr`
    - `setfiles`
    - `secon`
    - `genhomedircon`
    - `semodule`
    - `sestatus`
    - `setsebool`

## Others

- SMACK
- TOMOYO
- AppArmor
- Yama
- lockdown
- bpf

## Keyrings

- man keyrings
  - a thread-keyring is created only when a thread requests it
    - it has the same `_tid.`
  - a process-keyring is created only when a process requests it
    - it has the same `_pid.`
  - a session-keyring is usually created by `pam_keyinit` when a user logs in
    - it has the same `_ses.`
    - it is inherited across clone and fork so child processes (i.e., every
      process in the session) can manipulate the keyring
  - a user-keyring is usually also created by `pam_keyinit` when a user logs
    in
    - it has the same `_uid.`
    - all processes under the same real user can manipulate the keyring
    - `pam_keyinit` also links it to the seesion-keyring such that it can be
      searched
- when kernel or userspace need a key (e.g., when a thread accesses an
  encrypted ext4 directory), kernel searches, in this order,
  - thread-keyring
  - process-keyring
  - session-keyring
- `keyctl`
  - `keyctl show`
    - on normal Linux system, there will be a session keyring and a user keyring
      linked together
- kernel
  - `register_key_type` registers a new key type
    - `key_type_keyring` is special and is not registered this way
  - `keyring_alloc` allocates a key ring, which is a key of type `keyring`
  - `key_create` creates a key of the specified type in a keyring
