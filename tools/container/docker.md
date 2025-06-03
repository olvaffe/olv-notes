Docker
======

## Get started

- to run a container, `docker run -d -p 80:80 docker/getting-started`
  - `-d` means detached
  - `-p host:container` maps host port to container port
  - `docker/getting-started` is the image to use for the container
- a container is a process running in a jail
  - its rootfs is an overlayfs
  - the image is the lower filesystem and is ro
  - a temp directory, `/var/lib/docker/overlay2/<sha>/diff`, is the upper fs
    and is rw
  - when the container exits, all fs changes are gone
  - `docker container commit` can create a new image that merges the lower and
    the upper fs
- `docker container`
  - `docker container ls` lists running containers
  - `docker container ls -a` lists all containers
  - `docker container stop <id>` kill()s the container process
    - the states are still available at `/var/lib/docker/containers/<id>`
  - `docker container rm <id>` to remove the container
- to create a volume,
  - `docker volume create <name>`
  - the volume is at `/var/lib/docker/volumes/<name>`
- to mount a volume,
  - `docker run -v <name>:/path/to/mount some-image`
  - this bind-mounts `/var/lib/docker/volumes/<name>/_data` to `/path/to/mount`
    for the container
  - data written to `/path/to/mount` are persistent

## Build an Image

- `docker image build` builds an image from a `Dockerfile`
  - each command in `Dockerfile` creates an (intermediate) image
  - logically, each command, except for the first `FROM`,
    - create a container using the last image
    - execute the command
    - commit to create a new image
- multi-stage build

## Dockerfile

- <https://github.com/mbentley/docker-omada-controller/blob/master/Dockerfile.v5.x>
  - uses ubuntu 20.04 as the base image
  - `install.sh` downloads the software controller tarball, installs
    dependencies, unpacks the tarball
    - the dependencies are `ca-certificates`, `unzip`, `wget`, `gosu`,
      `net-tools`, `tzdata`, `mongodb-server-core`, and
      `openjdk-17-jre-headless`
    - it untars and copies selected files to to `/opt/tplink/EAPController`
  - `log4j_patch.sh` is nop for 5.x
  - `healthcheck.sh` is run every 5 minutes to make sure the controller is
    still alive
  - the container runs `ENTRYPOINT CMD`
    - `ENTRYPOINT` is `/bin/sh -c` by default and is overridden to
      `entrypoint.sh` in this case
    - `CMD` runs `java -server com.tplink.smb.omada.starter.OmadaLinuxMain`
  - <https://www.tp-link.com/us/support/faq/3281/>
    - app connects to controller tcp 8843 (`PORTAL_HTTPS_PORT`) for auth and
      tcp 8043 (`MANAGE_HTTPS_PORT`) for management
      - or to controller tcp 8088 (`PORTAL_HTTP_PORT` and `MANAGE_HTTP_PORT`)
        for both
    - eap connects to controller tcp 29814 (`PORT_MANAGER_V2`), tcp 29815
      (`PORT_TRANSFER_V2`), and tcp 29816 (`PORT_RTTY`)
      - eap on older firmware connect to tcp 29811, 29812, and 29813
    - eap powers on and broadcasts to udp 29810 (`PORT_DISCOVERY`) to announce
      itself
    - app initializes and broadcasts to udp 27001 (`PORT_APP_DISCOVERY`) to
      discover controller
    - controller connects to tcp 27217 for mongodb

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
