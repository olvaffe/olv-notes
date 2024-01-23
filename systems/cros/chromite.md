CrOS chromite
=============

## `depot_tools`

- <https://chromium.googlesource.com/chromium/tools/depot_tools.git>
- running a command can sliently `git pull` the repo
- `fetch` fetches pre-defined projects using `gclient`
- `gclient` is a meta-checkout tool
  - it parses `.gclient` which is set up by `fetch`
  - it also parses `DEPS` which is checked in to the project
- `git cl` interacts with gerrit

## Manifests

- repos
  - external: <https://chromium.googlesource.com/chromiumos/manifest>
  - internal: <https://chrome-internal.googlesource.com/chromeos/manifest-internal>
- branches of external manifest repo
  - `main ` is synced from internal `main` periodically
  - `snapshopt` adds a new "annealing" commit every 30 minutes
    - all projects have fixed revisions
    - each commit updates the revisions of the projects
    - this can be used to build a release image or a test image
  - `stable` is several hours behind `snapshot`
    - after an "annealing" commit is created, CQ images are built and tested
    - if all good, `stable` is updated
  - `postsubmit` is half day behind `snapshot`
    - once or twice a day, all images are built and tested
    - if all good, `postsubmit` is updated
- release builds
  - <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+log/refs/heads/main/chromeos/config/chromeos_version.sh>
    - once a day, `CHROMEOS_BUILD` is bumped and all images are built and
      tested

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

## Artifacts

- `Get`
  - `factory_image.zip`
  - `stripped-packages.tar`
  - `debug.tgz`
  - `debug_breakpad.tar.xz`
  - `sysroot_chromeos-base_chromeos-chrome.tar.xz`
  - `environment_chromeos-base_chromeos-chrome.tar.xz`
  - `chromeos-hwqual-*.tar.bz2`
- `BundleImageZip`
  - zip image dir into `image.zip`
- `BundleAutotestFiles`
  - generate `control_files.tar`
  - generate `autotest_packages.tar`
  - generate `autotest_server_package.tar.bz2`
  - generate `test_suites.tar.bz2`
- `BundleTastFiles`
  - generate `tast_bundles.tar.bz2`
- `BundleEbuildLogs`
  - generate `ebuild_logs.tar.xz`
- `BundleTestUpdatePayloads`
  - generate various files from `chromiumos_test_image.bin`
  - `*delta_dev.bin` and `stateful.tgz` for update
  - `full_dev_part_*.bin.gz` for `quick-provision` script
- `BundleFpmcuUnittests`
  - create `fpmcu_unittests.tar.bz2` for on-device fingerprint MCU unit tests
- `BundleChromeOSConfig`
  - copy `usr/share/chromeos-config/yaml/config.yaml/config.yaml` from sysroot
- `BundleImageArchives`
  - invokes `ArchiveImages` to tar
    - `chromiumos_base_image` into `chromiumos_base_image.tar.xz`
      - this and others are disk images
      - they are sparse; for example, ls reports 6.5G and du reports 2.1G
    - `chromiumos_test_image.bin` into `chromiumos_test_image.tar.xz`
    - `recovery_image.bin` into `recovery_image.tar.xz`
      - recovery image has a minimal stateful partition comparing to regular
        images
    - `vmlinuz.bin` into `vmlinuz.tar.xz`
- `BundleFirmware`
  - invokes `BuildFirmwareArchive` to build `firmware_from_source.tar.bz2`
    which contains all CR50/EC/AP firmwares
