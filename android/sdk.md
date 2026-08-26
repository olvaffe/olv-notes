# Android SDK

## Installation

- <https://developer.android.com/tools/agents/android-cli>
  - `curl -O https://dl.google.com/android/cli/latest/linux_x86_64/android`
  - `chmod +x android`
  - `./android update` downloads latest cli to `~/.android`
- sdk
  - SDK path defaults to `~/Android/Sdk`
    - export `ANDROID_HOME` to override
  - `./android sdk list --all` lists all packages
  - `./android sdk install <packages>...`
    - `platform-tools`, `ndk/<ver>`, `cmake/<ver>`, `build-tools/<ver>`,
      `platforms/android-<ver>`
- update
  - `./android update` self-updates
  - `./android sdk update` updates sdk packages
- if only adb is needed,
  - <https://developer.android.com/studio/releases/platform-tools>

## Packages

- `platform-tools` to communicate with devices
  - always use the latest version
  - `adb`
  - `fastboot`
- `build-tools/<ver>` to build Android apks
  - manual install needed if not using gradle
  - `aapt`
  - `aidl`
  - `d8`
- `platforms/android-<ver>` for various runtime versions
  - manual install needed if not using gradle
  - `android.jar`
- `ndk/<ver>` to build native binaries
  - manual install needed if not using gradle
- `cmake/<ver>` to build native binaries using cmake
  - manual install needed

## NDK

- <https://developer.android.com/ndk/guides/ndk-build>
  - e.g., <https://github.com/LunarG/gfxreconstruct/commit/27351d2978dfd4f2f934be6bbc4982bbc5099b9e>
    - `git clone https://github.com/lz4/lz4.git`
    - create `jni/Android.mk` and `jni/Application.mk`
    - `PATH=$ANDROID_NDK:$PATH ndk-build`

## Emulator

- use sdkmanager to install `system-images` first
- `avdmanager`
  - `list target` lists installed `platforms` versions
  - `list device` lists devices (with pre-configured emulated hw features?)
  - `create avd -n abc -k system-images;android-33;default;arm64-v8a`
    - this creates avd under `~/.android/avd`
  - `list avd` lists virtual devices
- `emulator`
