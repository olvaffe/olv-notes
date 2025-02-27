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

## Compose file reference

- <https://docs.docker.com/reference/compose-file/>
- `name` defines `COMPOSE_PROJECT_NAME`
- `services`
  - `<foo>` is the name of a service
  - `annotations` annotates the service
  - `attach` enables/disables service log collection
  - `build` specifies build config for image creation
  - `blkio_config` specifies block io limits
  - `cpu_count`, `cpu_percent`, `cpu_shares`, `cpu_period`, `cpu_quota`,
    `cpu_rt_runtime`, `cpu_rt_period`, `cpus`, `cpuset` specifies cpu limits
  - `cap_add` and `cap_drop` add/drop specified caps
  - `cgroup` and `cgroup_parent`
  - `command` overrides `CMD`
  - `configs` specifies configs to accessible by the service
  - `container_name` is the name of the container
  - `credential_spec` is for windows ad
  - `depends_on` controls startup and shutdown order between services
  - `deploy` specifies the deploy config
  - `develop` specifies the develop config
  - `device_cgroup_rules`
  - `devices` bind-mounts specified devices (`/dev/xxx`)
  - `dns`, `dns_opt`, and `dns_search` specify `/etc/resolv.conf`
  - `domainname` specifies the domain name
  - `driver_opts` overrides network driver ops
  - `entrypoint` overrides `ENTRYPOINT`
  - `env_file` and `environment` specifies envvars
  - `expose` exposes a port to other services but not to host
  - `extends` uses the specified service as the base
  - `external_links` refers to services that have been created externally
  - `extra_hosts` specifies `/etc/hosts`
  - `gpus` specifies gpu limits
  - `group_add` specifies the supplementary groups
  - `healthcheck` overrides `HEALTHCHECK`
  - `hostname` specifies the host name
  - `image` specifies the image
  - `init` runs pid 1
  - `ipc` specifies ipc isolation mode
  - `isolation` specifies platform-specific isolation tech
  - `labels` and `label_file` add metadata to the service
  - `links` adds network links to other services
    - by default, services are bridged and can access each other already
  - `logging` controls logging
  - `mac_address` specifies nic mac
  - `mem_limit`, `mem_reservation`, `mem_swappiness`, `memswap_limit` specify
    memory limits
  - `network_mode` and `networks` specify networks
  - `oom_kill_disable` and `oom_score_adj` controls oom killer
  - `pid` and `pids_limit` controls pids
  - `platform` specifies `os/arch/variant`
  - `ports` specifies host port mapping
  - `post_start` and `pre_stop` are lifecycle hooks
  - `privileged`
  - `profiles`
  - `pull_policy` specifies the image pull policy
  - `read_only` enables a ro filesystem
  - `restart` specifies the restart policy
  - `runtime` specifies the oci runtime
  - `scale`
  - `secrets` specifies secrets accessible by this service
  - `security_opt`
  - `shm_size` limits `/dev/shm`
  - `stdin_open` keeps stdin open
  - `stop_grace_period` is the wait period between `stop_signal` and `SIGKILL`
  - `stop_signal` defaults to `SIGTERM`
  - `storage_opt` specifies storage driver opts
  - `sysctls` customizes sysctl
  - `tmpfs` mounts specified paths as tmpfs
  - `tty` allocates a tty
  - `ulimits` overrides ulimits
  - `user` overrides `USER`
  - `userns_mode` specifies user namespace mode
  - `uts` specifies uts namespace mode
  - `volumes` specifies volumes accessible by this service
  - `volumes_from` mounts all volumes specified by another service
  - `working_dir` overrides `WORKDIR`
- `networks`
  - `<foo>` is the name of a network
  - `driver` specifies the driver
  - `driver_opts` specifies driver opts
  - `attachable` allows standalone containers to attach to the network
  - `enable_ipv4` enables/disables ipv4
  - `enable_ipv6` enables/disables ipv6
  - `external` means the network has been created externally
  - `ipam` is for ip address management
  - `internal` creates an isolated network (no internet access)
  - `labels` adds metadata to the network
  - `name` is the lookup name of the network
    - the compose file refers to the network by `<foo>`, but the storage engine
      uses this value to look up the network
- `volumes`
  - `<foo>` is the name of a volume
  - `driver` specifies the driver
  - `driver_opts` specifies driver opts
  - `external` means the volume has been created externally
  - `labels` adds metadata to the volume
  - `name` is the lookup name of the volume
- `configs`
  - `<foo>` is the name of a config, accessible at `/<foo>`
  - `file` uses the contents of a file for the config
  - `environment` uses the contents of an envvar for the config
  - `content` uses the inline value for the config
  - `external` means `/<foo>` has been created externally
  - `name` is the lookup name of the config
- `secrets`
  - `<foo>` is the name of a secret, accessible at `/run/secrets/<foo>`
  - `file` uses the contents of a file for the secret
  - `environment` uses the contents of an envvar for the secret
