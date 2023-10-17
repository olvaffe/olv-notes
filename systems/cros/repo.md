CrOS repos
=========

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
