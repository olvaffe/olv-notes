# Chrome OS hdctools

## `start-servod`

- `start-servod`
  - it checks if docker is up and user belongs to docker group
  - if `development_environment` contents have changed, it rebuilds the
    bootstrap image
    - `docker build -f Dockerfile.bootstrap -t servod-bootstrap ../development_environment`
    - the base is `python:3.11-slim-bookworm`, debian bookworm plus python 3.11
    - `checksum` is the prev checksum to detect content change
  - it starts the bootstrap container to run `start_servod.py`
- `start_servod.py`
  - `parse_args`
    - `channel` defaults to `local`
    - `container_name`, `board`, `model`, `mount`, etc has no default
  - `get_image`
    - because channel defaults to `local`, it looks for `servod:dev` and falls
      back to `us-docker.pkg.dev/chromeos-hw-tools/servod/servod:release`
  - `start_servod`
    - it starts `servod` container to run `start_servod_dev.sh`
      - the args are `--port 9999 --board <board> --model <model>`
  - it exits and the bootstrap container is removed
- `start_servod_dev.sh`
  - it updates usb hub fw on servo if necessary
  - it starts the grpc server
    - `DriverService` talks to the dut
    - `SystemConfig` parses xmls
  - it starts `servod`
    - `ServoService` monitors the grpc server
  - `dut-control` will talk to `servod` which forwards the calls to grpc
