Android CDD
===========

## Vulkan Requirements

- terms
  - CDD: Compatibility Definition Document
    - defines the android api for app devs
    - ensures app compat across android devices
  - VSR: Vendor Software Requirements
    - defines the kernel and hal interfaces
    - ensures AOSP system images work across android devices
  - GMS: Google Mobile Services
    - a collection of proprietary apps and services from google
    - oem must sign MADA (Mobile Application Distribution Agreement) to
      license
  - GMS Requirements
    - defines requirements a device must comply to preload GMS
    - enforced by MADA
  - CTS: Compatibility Test Suite
    - tests CDD compliance
  - VTS: Vendor Test Suite
    - tests VSR compliance
  - GTS: GMS Test Suite
    - tests GMS compliance
- Vulkan features
  - <https://developer.android.com/reference/android/content/pm/PackageManager>
  - `FEATURE_VULKAN_HARDWARE_VERSION`
    - 1.1 implies
      - `VK_ANDROID_external_memory_android_hardware_buffer`
      - `SYNC_FD`
      - `samplerYcbcrConversion`
    - may be software-based
  - `FEATURE_VULKAN_HARDWARE_LEVEL`
    - 0 implies `textureCompressionETC2`
    - 1 implies
      - `textureCompressionASTC_LDR`
      - and many more
  - `FEATURE_VULKAN_HARDWARE_COMPUTE`
    - 0 implies `VK_KHR_variable_pointers` and more
- Android Q+ requires Vulkan 1.1
  - <https://android-developers.googleblog.com/2019/05/whats-new-in-android-q-beta-3-more.html>
  - that's probably required by GTS/VTS for new 64-bit devices
- Android 13 CDD
  - <https://source.android.com/docs/compatibility/13/android-13-cdd>
  - strongly recommend Vulkan 1.3
  - strongly recommend Android Baseline 2021 profile

