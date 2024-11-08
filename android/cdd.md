Android CDD
===========

## Terms

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

## API Levels

- <https://source.android.com/docs/core/architecture/api-flags>
- system partition
  - `ro.product.first_api_level` is the initial sdk api level of the system
    partition
    - it never changes once the product ships
  - `ro.build.version.sdk` is the current sdk api level of the system
    partition
  - `ro.llndk.api_level` is the vendor api level of LLNDK
    - LLNDK is provided by the system partition
    - the vendor partition depends on LLNDK and must have a vendor api level
      equal to or lower than the vendor api level of LLNDK
- vendor partition
  - `ro.board.first_api_level` is the initial vendor api level of the vendor
    partition
    - it never changes once the soc is qualified for vendor freeze
  - `ro.board.api_level` is the current vendor api level of the vendor
    partition
  - `ro.vendor.api_level` is derived

## Vulkan

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
- Vulkan profiles
  - <https://github.com/KhronosGroup/Vulkan-Profiles/tree/main/profiles>
  - baseline profiles reflect the common denominator of vulkan impls
    - `VP_ANDROID_baseline_2021`
    - `VP_ANDROID_baseline_2022`
  - minimums profiles reflect the direction of vulkan impls
    - `VP_ANDROID_15_minimums`
    - `VP_ANDROID_16_minimums`
- Android Q+ requires Vulkan 1.1
  - <https://android-developers.googleblog.com/2019/05/whats-new-in-android-q-beta-3-more.html>
  - that's probably required by GTS/VTS for new 64-bit devices
- Android 13 CDD
  - <https://source.android.com/docs/compatibility/13/android-13-cdd>
  - strongly recommend Vulkan 1.3
  - strongly recommend Android Baseline 2021 profile
- Android 14 CDD
  - <https://source.android.com/docs/compatibility/14/android-14-cdd>
  - must support Android Baseline 2021 profile
  - strongly recommend Android Baseline 2022 profile
  - strongly recommend protected memory and global priority
  - strongly recommend skiavk for hwui
- Android 15 CDD
  - <https://source.android.com/docs/compatibility/15/android-15-cdd>
  - no change