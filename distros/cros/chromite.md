Chromite
========

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
