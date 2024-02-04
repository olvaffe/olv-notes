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
      - not working here...
    - `buildah commit alpine-working-container alpine-modified`
- <https://github.com/containers/youki>
  - it is a rust implementation of the runtime spec
- <https://github.com/containers/crun>
  - it is a c implementation of the runtime spec
- <https://github.com/containers/netavark>
- <https://github.com/containers/conmon>
- <https://github.com/containers/bubblewrap>

## Podman

- <https://github.com/containers/podman>
- `podman run`
  - `-i` keeps stdin open
  - `-t` allocates pty
  - `-e NAME=VAL` sets an envvar
  - `-p host_port:container_port` maps host port to container port
  - `-v host_dir:container_dir` creates a bind-mount

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
