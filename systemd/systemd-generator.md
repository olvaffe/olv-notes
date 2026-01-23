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
