Chromium chrome/android
=======================

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

