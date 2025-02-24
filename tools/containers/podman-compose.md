Podman Compose
==============

## Overview

- `podman compose` invokes either `docker-compose` or `podman-compose`
  - both scripts are written in python

## Systemd Integration

- as root,
  - `podman compose systemd -a create-unit` generates
    `/etc/systemd/user/podman-compose@.service`
  - `loginctl enable-linger <pod-user>` makes the pod user linger
- as pod user,
  - create `podman-composer.yml` for the pod
  - `podman compose systemd -a register` generates
    `~/.config/containers/compose`, pointing to `podman-composer.yml` in the
    current directory
  - `systemctl --user enable --now podman-compose@kavita.service` enables and
    starts the user service
