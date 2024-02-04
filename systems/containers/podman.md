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
- <https://github.com/opencontainers/runc> is the reference implementation of
  the runtime spec
