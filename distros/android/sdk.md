Android SDK
===========

## SDK/NDK

- Bootstrap to `~/android/sdk`
  - <https://developer.android.com/studio>
  - choose "Command line tools only"
  - `unzip commandlinetools-linux-*_latest.zip`
  - `./cmdline-tools/bin/sdkmanager --sdk_root=$HOME/android/sdk cmdline-tools\;latest`
  - `rm -rf commandlinetools-linux-*_latest.zip cmdline-tools`
- Install packages
  - `cd ~/android/sdk`
  - `./cmdline-tools/latest/bin/sdkmanager --list`
  - `./cmdline-tools/latest/bin/sdkmanager --install 'build-tools;33.0.0' 'ndk;25.0.8775105' platform-tools`
- if only adb is needed,
  - search for "android sdk platform tools" for direct download
  - <https://developer.android.com/studio/releases/platform-tools>

## Components

- `cmdline-tools` to manage SDK
  - `sdkmanager`
  - `avdmanager`
- `platform-tools` to communicate with devices
  - `adb`
  - `fastboot`
- `build-tools` to build Android apks
  - `aapt`
  - `aidl`
  - `d8`
- `ndk` to build native libraries
- `platforms` to various runtime versions
  - `android.jar`
- `emulator` is the emulator

## Emulator

- use sdkmanager to install `system-images` first
- `avdmanager`
  - `list target` lists installed `platforms` versions
  - `list device` lists devices (with pre-configured emulated hw features?)
  - `create avd -n abc -k system-images;android-33;default;arm64-v8a`
    - this creates avd under `~/.android/avd`
  - `list avd` lists virtual devices
- `emulator`
