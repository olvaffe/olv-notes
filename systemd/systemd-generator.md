systemd-generator
=================

## Overview

- `man systemd.generator`
- after systemd loads configs and before it loads unit files, systemd invokes
  all generators
  - this gives generators a chance to generate unit files to
    - system: `/run/systemd/generator`
    - user: `$XDG_RUNTIME_DIR/systemd/generator`
- this happens on `systemd daemon-reload` as well
  - the old generated unit files are cleaned up

## Generators

- `systemd-tpm2-generator`
  - if uefi advertises tpm support, it generates
    `/run/systemd/generator/sysinit.target.wants/tpm2.target` to wait for
    `/dev/tpm0`
  - if kernel lacks tpm driver, add `systemd.tpm2_wait=0` to kernel cmdline to
    disable the behavior
