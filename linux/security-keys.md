Kernel Keyrings
===============

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

## `keyctl`

- `keyctl show`
  - on normal Linux system, there will be a session keyring and a user keyring
    linked together
