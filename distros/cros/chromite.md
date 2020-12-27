Chromite
========

## `cros` CLI

- `bin/cros` is a symbolic link to `scripts/wrapper3.py` 
  - many under `bin/` are symlinks to `scripts/wrapper3.py`
- when invoked as `cros`, `scripts/wrapper3.py` finds `scripts/cros.py` and
  calls the `main` function
- `scripts/cros.py` runs a subcommmand under `cli/cros`

## `cros workon`

- `cli/cros/cros_workon.py`
  - it runs in chroot, and will automatically chroot if not
- its config is stored at `/mnt/host/source/.config/cros_workon`

## Scripts

- use `cros_sdk` as an example
  - people usually have `depot_tools` in their $PATH
  - when `cros_sdk` in `depot_tools` is run, it finds `cros_sdk` in
    `chromite/bin`
  - `chromite/bin/cros_sdk` runs `chromite/scripts/cros_sdk.py`
- once in SDK, `chromite/bin` is in $PATH

## cbuildbot

- chrome os build bot
- `ManifestVersionedSyncStage`
  - it calls `GetNextBuildSpec` which calls `GetNextVersion` and
    `PublishManifest`
  - `GetNextVersion` increments the current version and pushes a new commit
    - `VersionInfo` parses `VERSION_FILE`, which is
      `third_party/chromiumos-overlay/chromeos/config/chromeos_version.sh`
  - `PublishManifest` generates a manifest for the build and commit it to
    `https://chromium.googlesource.com/chromiumos/manifest-versions/`
