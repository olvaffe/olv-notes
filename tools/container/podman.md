Podman
======

## OCI

- <https://opencontainers.org/>
  - Open Container Initiative
- <https://github.com/opencontainers/distribution-spec> is the distribution
  spec
  - it defines a protocol for the distribution of content
  - Pull - Clients are able to pull from the registry
    - `GET /v2/<name>/manifests/<reference>` retrieves a manifest
      - such as an image index defined by the OCI image spec
    - `GET /v2/<name>/blobs/<digest>` retrieves a blob
  - Push - Clients are able to push to the registry
    - `POST /v2/<name>/blobs/uploads/` uploads a blob
    - `PUSH /v2/<name>/manifests/<reference>` uploads a manifest
  - Content Discovery - Clients are able to list or otherwise query the
    content stored in the registry
    - `GET /v2/<name>/tags/list` retrieves tags
  - Content Management - Clients are able to control the full life-cycle of
    the content stored in the registry
    - `DELETE /v2/<name>/manifests/<tag>` deletes a tag
    - `DELETE /v2/<name>/blobs/<digest>` deletes a blob
  - there are more api endpoints
- <https://github.com/opencontainers/image-spec> is the image spec
  - image layout
    - `oci-layout` is a json file with `"imageLayoutVersion": "1.0.0"`
    - `index.json` is the image index with an array of `manifests`
      - they provide digests of the manifests, which are stored as blobs
    - `blobs/<alg>/<encoded>`, many blobs
  - manifest
    - `config` provides the digest of the config stored as a blob
    - `layers` provides the digests of the layers stored as blobs
      - each layer adds/removes/modifies files of the prior layer
      - the runtime is expected to unpack the layers into a directory called a
        filesystem bundle
  - config
    - `created`, `author`, `history`, etc. are optional metadata
    - `architecture`, `os`, etc. are required metadata
    - `config` is optional and describes the execution parameters (uid, env,
      workdir, etc.).
    - `rootfs` is required
    - the runtime is expected to convert this to `config.json`
  - layer
- <https://github.com/opencontainers/runtime-spec> is the runtime spec
  - runtime must support
    - `state <container-id>` shows the state of the container
    - `create <container-id> <path-to-bundle>` creates a container according
      to its `config.json`
      - changes to `config.json` after this have no effect
    - `start <container-id>` starts the container
    - `kill <container-id> <signal>` sends a signal to the container
    - `delete <container-id>` deletes the container
  - container lifecycle
    - user invokes `create`
      - runtime creates the container environment according to `config.json`
      - runtime invokes the deprecated `prestart` hook
      - runtime invokes the `createRuntime` hook after namespaces are created
        - to customize namespaces
      - runtime invokes the `createContainer` hook after entering the mount
        namespace
        - to add mounts
    - user invokes `start`
      - runtime invokes the `startContainer` hook
      - runtime runs the program specified by `process`
      - runtime invokes the `postStart` hook
    - user invokes `delete`
      - runtime deletes the container environment
      - runtime invokes the `postStop` hook
  - `config.json`
    - `ociVersion` is the container version
    - `root.path` is an absoluate path or a relative path to the bundle,
      usually `rootfs`
    - `mounts` is an array of mountpoints
    - `process` is the program to run
      - `user` is the uid/gid/umask
    - more
- <https://github.com/opencontainers/runc> is the reference implementation of
  the runtime spec

## Tools

- <https://github.com/containers/skopeo>
  - it is a client for the distribution spec
  - it is a frontend for <https://github.com/containers/image>
  - `skopeo inspect docker://docker.io/library/busybox`
  - `skopeo inspect --config docker://docker.io/library/busybox`
  - `skopeo copy docker://docker.io/library/busybox dir:my-dir`
    - `man containers-transports` for different transports
- <https://github.com/containers/buildah>
  - it creates images following the image spec
  - `buildah images` lists images
  - `buildah containers` lists containers
  - to create an image modified from `alpine`
    - `alpine` is an alias for `docker.io/library/alpine`
      - `/etc/containers/registries.conf.d/00-shortnames.conf`
    - `buildah from alpine`
      - it pulls the base image from the registry
      - it creates a container named `alpine-working-container`
    - `buildah run alpine-working-container sh`
    - `buildah commit alpine-working-container alpine-modified`
- <https://github.com/containers/youki>
  - it is a rust implementation of the runtime spec
- <https://github.com/containers/crun>
  - it is a c implementation of the runtime spec
- <https://github.com/containers/podman>
- <https://github.com/containers/storage>
  - on debian, `containers-storage` must be installed to default to `overlay`
- <https://github.com/containers/netavark>
- <https://github.com/containers/conmon>
- <https://github.com/containers/podman-compose>
- <https://github.com/containers/bubblewrap>

## `podman container`

- <https://docs.podman.io/en/latest/markdown/podman-container.1.html>
- management
  - `list` or `ps` lists containers
  - `port` lists port mappings
  - `prune` removes non-running containers
  - `stats` shows resource usage of containers
  - `rename <CONTAINER> <NAME>` renames a container
  - `update <CONTAINER>` updates cgroup configuration
  - `inspect <CONTAINER>` shows the config of a container
- scripting
  - `wait <CONTAINER>` waits for a container to stop
  - `exists <CONTAINER>` returns 0 if a container exists
- lifecycle
  - `create <IMAGE> [COMMAND...]` creates a container from `<IMAGE>`
    - status is `created`
  - `start <CONTAINER>` starts a container
    - status is `running`
    - the main process is started in the background by default
  - `stop <CONTAINER>` stops a container
    - status is `exited`
    - it sends `SIGTERM` to the main process, followed by `SIGKILL` if the
      main process does not terminate in 10 seconds
  - `rm <CONTAINER>` removes a container
    - if a container is stuck in `stopping` status forever, `rm -f` to force
      removal
- other lifecycle
  - `init <CONTAINER>` initializes a created container
    - status is `initialized`
    - this does everything except for starting the main process
      - e.g., vfs mounted, `conmon`/`crun` running
  - `cleanup <CONTAINER>` undoes `init`
    - status is back to `created`
  - `pause <CONTAINER>` pauses a running container
    - status is `paused`
    - this uses freezer cgroup or `SIGSTOP`
  - `unpause <CONTAINER>` undoes `pause`
  - `restart` stops and starts a running container
  - `kill <CONTAINER>` sends a signal to the main process of a running
    container
  - `clone <CONTAINER>` creates a container from `<CONTAINER>`
  - `run <IMAGE>` creates and starts a container from `<IMAGE>`
  - `runlabel <LABEL> <IMAGE>` runs a labeled command on the host
    - e.g., the command might be `podman run <IMAGE>` plus necessary args
- rootfs
  - `diff <CONTAINER>` shows the diff with the parent layer
  - `commit <CONTAINER>` creates an image from the current rootfs
  - `cp` copies files between host and containers
  - `mount <CONTAINER>` mounts the rootfs for host access
  - `unmount` undoes `mount`
  - `export <CONTAINER>` tars the rootfs
- execution
  - `attach <CONTAINER>` attaches to the main process
    - `start` by default starts the main process in the background
    - this subcommand attaches to the main process
  - `exec <CONTAINER> [COMMAND...]` executes a command inside a started
    container
  - `logs` shows the stdout/stderr of the main process
    - by default, the stdout/stderr of the main process are pipes owned by the
      runtime
    - this subcommand shows the contents
  - `top <CONTAINER>` shows the processes in the container
  - `checkpoint` snapshots the current state of a container
  - `restore` restores the state of a container
- `create` args
  - `--attach` or `-a` is for??
  - `--device` binds a host device node into the container
  - `--env` or `-e` sets an environment variable
    - this affects all processes (the main one, and those started with `exec`)
      spawned by the runtime
  - `--init` binds mount `catatonit` to `/run/podman-init` and starts it as
    pid 1
  - `--interactive` or `-i` keeps stdin open
    - `podman run busybox ls -l /proc/self/fd` says
      - stdin is `/dev/null`
      - stdout and stderr are pipes
    - `podman run -i busybox ls -l /proc/self/fd` says
      - stdin, stdout, and stderr are pipes
  - `--name` sets the name for the container
    - otherwise, a randomly generated name is used
  - `--network <MODE>` specifies the network mode
    - `slirp4netns` is the default for rootless containers
    - `bridge` is the default for rootful containers
      - all rootful containers in this mode are on the same bridge
    - `host` shares the network namespace with the host
  - `--privileged` gives the container the same access as the user launching
    the container
  - `--publish` or `-p host_port:container_port` maps host port to container
    port
  - `--pull never` fails creation when the image is missing
  - `--restart <POLICY>` specifies what to do when the main process terminates
  - `--rm` removes the container on stop
  - `--stop-timeout` sets the stop timeout
    - by default, `stop` or `rm -f` sends `SIGTERM`, waits for 10s, and sends
      `SIGKILL`
    - this overrides the default stop timeout for the container
  - `--tty` or `-t` allocates pty
    - `podman run -it busybox ls -l /proc/self/fd` says
      - stdin, stdout, and stderr are `/dev/pts/0`
  - `--tz <TZ>` bind-mounts the timezone to `/etc/localtime`
    - `local` to bind-mount host timezone
    - some guides set `TZ` instead, which is parsed by glibc
  - `--user` or `-u` specifies the uid of all processes spawned by the runtime
    in the container
    - otherwise, the runtime uses what's specified by the image which is
      usually 0
  - `--volume` or `-v host_dir:container_dir` creates a bind-mount
  - `--workdir` or `-w` specifies the initial directory of all processes
    spawned by the runtime

## `podman image`

- <https://docs.podman.io/en/latest/markdown/podman-image.1.html>
  - `build` builds an image using a `Containerfile` or `Dockerfile`
    - this uses code from `buildah`
  - `diff` shows delta between the last two layers of an image, or between two
    images
  - `exists` returns 0 if an image exists locally
  - `history` shows info about layers of an image
  - `import` imports an image from a tarball
    - `podman container export` can create a tarball
    - the imported image has a single layer
  - `inspect` shows the config of an image
  - `list` lists local images
  - `load` loads an image from a tarball
    - `podman image save` can create a tarball
    - the image may have multiple layers
  - `mount` mounts an image for host access
  - `prune` removes all unused images
  - `pull` pulls an image from a registry
  - `push` pushes an image to a registry
  - `rm` removes an image
  - `save` saves an image to a tarball
  - `scp` copyes an image between hosts or between storages
  - `search` searches a registry for images
  - `sign` creates a signature for an image
  - `tag` adds a new name for an image
    - this creates a new image pointing to the same blob
  - `tree` shows layers of an image in a tree format
  - `trust` manages registry trust policy
  - `unmount` undoes `mount`
  - `untag` undoes `untag`

## `podman compose`

- `podman compose` invokes either `docker-compose` or `podman-compose`
  - both scripts are written in python
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

## Quadlet systemd integration

- create a user for each rootless container
  - `useradd -m -s /bin/bash pod-foo`
  - optionally `-b /var/lib`
- start the user systemd instance on boot
  - `loginctl enable-linger pod-foo`
- pull the image beforehand
  - `podman pull <IMAGE>`
- create `~/.config/containers/systemd/foo.container`
  - `[Container]`
  - `Image=bar`
  - `Timezone=local`
  - `Volume=/source:/dest`
  - `PublishPort=8080:8080`
  - `StopTimeout=60`
  - `[Service]`
  - `TimeoutSec=70`
  - `[Install]`
  - `WantedBy=default.target`
- test
  - `/usr/lib/systemd/system-generators/podman-system-generator --user --dryrun`
  - `systemctl --user daemon-reload` to generate and reload
    `$XDG_RUNTIME_DIR/systemd/generator/foo.service`
- before podman 4.4,
  - `podman container create --name foo <IMAGE> <COMMAND>`
  - `podman generate systemd --name foo --new --restart-policy always \
       --start-timeout 600 --stop-timeout 60 > \
       ~/.config/systemd/user/container-foo.service` to generate the unit
  - `systemctl --user daemon-reload`
  - `systemctl --user enable --now container-foo`
