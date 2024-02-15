sudo
====

## Overview

- `man sudoers`
  - `%wheel ALL=(ALL:ALL) NOPASSWD: ALL` means
    - group `wheel` on `ALL` hosts
    - can run as `ALL` users and `ALL` groups
    - without password
    - `ALL` commands
  - `env_reset` to reset the environment to minimal
  - `env_keep` to preserve specified environment variables
  - `timestamp_type` to change how per-user credential is cached
  - `timestamp_timeout` minutes before sudo asks for password again
- cmdline
  - `sudo VAR=VAL cmd` to specify additional environment variables
  - `sudo -E` to preserve the environment
