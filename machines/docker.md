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
