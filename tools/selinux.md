SELinux
=======

## Setup

- enable `pam_selinux.so` to set up
  - execution security context for the following `execve`
  - file security context for the controlling terminal
  - security context for new kernel keyring
- `make bare` cleans up
- `make conf` generates `policy/booleans.conf` and `policy/modules.conf`
- `make install`

## Reference Policy

- <https://github.com/SELinuxProject/refpolicy.git>
- `m4` processes the source files
  - `policy/users` to `system.users`
