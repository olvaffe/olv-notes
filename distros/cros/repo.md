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
    - I guess it stays at the latest tested-good snapshot
