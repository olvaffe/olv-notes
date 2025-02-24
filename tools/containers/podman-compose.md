Podman Compose
==============

## Overview

- `podman compose` invokes either `docker-compose` or `podman-compose`
  - both scripts are written in python

## Systemd Integration

- as root,
  - `podman compose systemd -a create-unit` generates
    `/etc/systemd/user/podman-compose@.service`
  - if using `/var` for pod users, `useradd -r -b /var/lib -m -F -s /bin/bash
    <pod-user>` creates the pod user
  - `loginctl enable-linger <pod-user>` makes the pod user linger
- as pod user,
  - create `podman-composer.yml` for the pod
  - `podman compose systemd -a register` generates
    `~/.config/containers/compose`, pointing to `podman-composer.yml` in the
    current directory
  - `systemctl --user enable --now podman-compose@<pod-user>.service` enables and
    starts the user service
