# Android APEX

## Overview

- <https://source.android.com/docs/core/ota/apex>
  - an apex file is an apk file under
    - `/system/apex`
    - `/vendor/apex`
    - `/data/apex`
  - an apex file has `apex_payload.img` that is uncompressed and can be
    mounted directly
    - apexd creates `/apex/<name>@<version>`
    - it mounts `apex_payload.img` to the dir using `fsoffset` mount option
    - it bind-mounts the highest version to `/apex/<name>`
- `apex_payload.img`
  - `app` provides apks
    - this mechanism is used by low-level apk that depends on other system
      components packaged in the apex
  - `bin` provides native binaries and scripts
  - `etc` provides config files
    - `init/*.rc` provides updated services
  - `javalib` provides jar files
    - they can be isolated to the apex or be made globally available
  - `lib64` provides native libraries for app, bin, and javalib
    - they can be isolated to the apex or be made globally available
  - `priv-app` provides privileged apks
