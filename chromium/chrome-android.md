# Chromium chrome/android

## Packages

- `com.google.android.webview` has standalone and trichrome variants
  - standalone variant has `libwebviewchromium.so`
  - trichrome variant refers to trichrome lib
- `com.android.chrome` only has trichrome variant
  - it refers to trichrome lib
  - it has `libelements.so` for ui
- `com.google.android.trichromelibrary_<ver>`
  - it has `libmonochrome_64.so`
- other channels
  - `com.chrome.dev` is beta-channel chrome app
    - it is always standalone and has `libchrome.so`
  - `com.chrome.dev` is dev-channel chrome app
    - it is always standalone and has `libchrome.so`
  - `com.chrome.canary` is canary-channel chrome app
    - it is always standalone and has `libchrome.so`
- `adb shell dumpsys package`
  - `Package [com.android.chrome] (...):`
    - `codePath=/product/app/Chrome`
      - this is the bundled chrome app
    - `versionName` gives the version
    - `usesStaticLibraries` gives the trichrome lib version
    - `usesLibraryFiles` gives the trichrome lib apk
  - `Package [com.google.android.trichromelibrary_...] (...):`
    - `codePath=/product/app/TrichromeLibrary`
      - this is the bundled trichrome lib
  - `Package [com.google.android.webview] (...):`
    - `codePath=/product/app/WebViewGoogle`
      - this is the bundled webview
      - this is standalone variant
  - `Package [com.android.chrome] (...):`
    - `codePath=/data/app/...`
      - this is the updated chrome app
  - `Package [com.google.android.trichromelibrary_...] (...):`
    - `codePath=/data/app/...`
      - this is the updated trichrome lib

## Android

- adb
  - `adb shell am start -n com.android.chrome/com.google.android.apps.chrome.Main`
  - `adb shell am start -n com.android.chrome/org.chromium.chrome.browser.ChromeTabbedActivity -d "https://www.example.com"`
- features
  - `gpu/config/gpu_finch_features.cc`
    - `kVulkan` is `FEATURE_ENABLED_BY_DEFAULT`
  - `ui/gl/gl_switches.cc`
    - `kDefaultANGLEVulkan` is `FEATURE_DISABLED_BY_DEFAULT`
    - `kVulkanFromANGLE` is `FEATURE_DISABLED_BY_DEFAULT`
