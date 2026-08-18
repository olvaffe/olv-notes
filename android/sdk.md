# Android SDK

## Installation

- <https://developer.android.com/tools/agents/android-cli>
  - `curl -O https://dl.google.com/android/cli/latest/linux_x86_64/android`
  - `chmod +x android`
  - `./android --sdk ~/android/sdk sdk install <packages>...`
    - `platform-tools`, `ndk`, `cmake`, `build-tools`
    - `./android sdk list --all` lists all packages
- SDK path defaults to `~/Android/Sdk`
  - export `ANDROID_HOME` to override
- update
  - `./android update` self-updates
    - it downloads latest cli to `~/.android`
  - `./android sdk update` updates sdk packages
- if only adb is needed,
  - <https://developer.android.com/studio/releases/platform-tools>

## Packages

- `platform-tools` to communicate with devices
  - always use the latest version
  - `adb`
  - `fastboot`
- `build-tools/<ver>` to build Android apks
  - `aapt`
  - `aidl`
  - `d8`
- `platforms/android-<ver>` for various runtime versions
  - `android.jar`
- `ndk/<ver>` to build native binaries
- `cmake/<ver>` to build native binaries using cmake

## Emulator

- use sdkmanager to install `system-images` first
- `avdmanager`
  - `list target` lists installed `platforms` versions
  - `list device` lists devices (with pre-configured emulated hw features?)
  - `create avd -n abc -k system-images;android-33;default;arm64-v8a`
    - this creates avd under `~/.android/avd`
  - `list avd` lists virtual devices
- `emulator`
