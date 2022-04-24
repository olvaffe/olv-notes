systemd-nspawn
==============

## Basics

- see <distros/disk.md> to create a chroot or disk image with `mkosi`
  - or it can be manual
- `sudo systemd-nspawn -D <chroot>` to start shell as pid 1
  - `--boot` searches init and starts it as pid 1 instead
  - if starting a command that is not init or shell, specify `-a` as well
    - this starts a stub init as pid 1, to reap processes
- namespaces
  - user namespace is disabled unless `-U`
