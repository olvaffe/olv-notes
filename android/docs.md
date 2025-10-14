Android Docs
============

## CDD Terms

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
    - aka ABP, android baseline profile
  - minimums profiles reflect the direction of vulkan impls
    - `VP_ANDROID_15_minimums`
    - `VP_ANDROID_16_minimums`
    - aka VPA, vulkan profile for android
- Android 10 CDD
  - <https://source.android.com/docs/compatibility/10/android-10-cdd>
  - strongly recommend Vulkan 1.1
  - <https://android-developers.googleblog.com/2019/05/whats-new-in-android-q-beta-3-more.html>
    - that's probably required by GTS/VTS for new 64-bit devices
- Android 11 CDD
  - <https://source.android.com/docs/compatibility/11/android-11-cdd>
  - `graphics.gpu.profiler.support`
    - `GpuCounterEvent`
    - `GpuRenderStageEvent`
    - `GpuFrequencyFtraceEvent`
      - `(gpu, freq)`
    - no mention of `GpuMemTotalFtraceEvent`
      - `(gpu, pid, total mem)`
- Android 12 CDD
  - <https://source.android.com/docs/compatibility/12/android-12-cdd>
- Android 13 CDD
  - <https://source.android.com/docs/compatibility/13/android-13-cdd>
  - strongly recommend Vulkan 1.3
  - strongly recommend Android Baseline 2021 profile
  - `GpuWorkPeriodFtraceEvent`
    - `(gpu, uid, start, end, active)`
- Android 14 CDD
  - <https://source.android.com/docs/compatibility/14/android-14-cdd>
  - must support Android Baseline 2021 profile
  - strongly recommend Android Baseline 2022 profile
  - strongly recommend protected memory and global priority
  - strongly recommend skiavk for hwui
- Android 15 CDD
  - <https://source.android.com/docs/compatibility/15/android-15-cdd>
  - no change
- Android 16 CDD
  - <https://source.android.com/docs/compatibility/16/android-16-cdd>
  - must support Android Baseline 2021 profile
  - must support Vulkan 1.1
  - strongly recommend Vulkan 1.3
  - strongly recommend Android Baseline 2022 profile
  - strongly recommend `protectedMemory` and `VK_EXT_global_priority`
  - strongly recommend `SkiaVk` with hwui
- VSR
  - strongly recommend becomes must?
  - vulkan 1.3
  - ABP 2021, 2022, etc.
  - VPA 15, 16, etc.
  - `SkiaVk` for hwui and re

## Development

- In Google, there is an internal master branch
  - Long term developments happen on the branch
  - AOSP contributions go to the branch
  - Very unstable
- A named branch is created from master months before a release
  - e.g. `gingerbread`
  - To stabilize
- A -release branch is also created with the named branch
  - e.g. `gingerbread-release`
  - Used to build official releases (e.g., SDK images)
- When the time to open source comes,
  - `gingerbread-release` is filtered to pushed out
  - so does `gingerbread`
- the new public `gingerbread` is merged to public `master`
- With that rationale in mind, take `build/` for example
  - `gingerbread` and `gingerbread-release` were released
  - `gingerbread` was merged to `master`, which was then froyo-based
  - AOSP `master` recevies updates continuously
    - fixes, features for next major release
  - AOSP `gingerbread` receives updates occasionally
    - fixes, features for next minor release
  - AOSP `gingerbread-release` receives updates occasionally
    - just fixes.
    - When it is time to do a minor release, `gingerbread` is
      partially merged.  Features not ready may be omitted.
  - the build id for
    - `master` is `OPENMASTER`
    - `gingerbread` is `GINGERBREAD`
    - `gingerbread-release` is a real id
  - official images are built from `gingerbread-release` (later
    `gingerbread-mr4-release`)
    - release tags are created from this branch
- Which branch to use?
  - current minor release: `android-<version>_<revision>`
  - current minor release plus fixes: `gingerbread-mr4-release`
  - possible next minor release: `gingerbread`
  - bleeding edge: `master`

## SELinux

- <https://source.android.com/docs/security/features/selinux>
- `file_contexts` labels files
  - `/vendor/lib(64)?/hw/vulkan\.intel\.so u:object_r:same_process_hal_file:s0`
- e.g., graphics composer
  - `system/sepolicy/public`
    - `hal_attribute(graphics_composer);` expands to
      - `attribute hal_graphics_composer;`
      - `attribute hal_graphics_composer_client;`
      - `attribute hal_graphics_composer_server;`
    - `attribute hal_graphics_composer_client_tmpfs;`
    - `type hal_graphics_composer_server_tmpfs, file_type;`
    - `type hal_graphics_composer_hwservice, hwservice_manager_type, protected_hwservice;`
    - `type hal_graphics_composer_service, protected_service, hal_service_type, service_manager_type;`
  - `system/sepolicy/vendor`
    - `type hal_graphics_composer_default, domain;`
    - `hal_server_domain(hal_graphics_composer_default, hal_graphics_composer)` expands to
      - `typeattribute hal_graphics_composer_default halserverdomain;`
      - `typeattribute hal_graphics_composer_default hal_graphics_composer_server;`
      - `typeattribute hal_graphics_composer_default hal_graphics_composer;`
    - `type hal_graphics_composer_default_exec, exec_type, vendor_file_type, file_type;`
    - `init_daemon_domain(hal_graphics_composer_default)` expands to
      - rules to allow doman transition
      - `type_transition init hal_graphics_composer_default_exec:process hal_graphics_composer_default;`
        - when init starts `hal_graphics_composer_default_exec`, transition to
          `hal_graphics_composer_default` automatically
    - `type_transition hal_graphics_composer_default tmpfs:file hal_graphics_composer_server_tmpfs;`
    - `allow hal_graphics_composer_default hal_graphics_composer_server_tmpfs:file { getattr map read write relabelfrom };`
  - `device/<vendor>/<device>-sepolicy`
    - `/vendor/bin/hw/android\.hardware\.composer\.hwc3-service\.drm u:object_r:hal_graphics_composer_default_exec:s0`
  - `system/sepolicy/private`
    - `binder_call(hal_graphics_composer_client, hal_graphics_composer_server)`
      - client can call to server
    - `binder_call(hal_graphics_composer_server, hal_graphics_composer_client)`
      - server can call back to client
    - `hal_attribute_hwservice(hal_graphics_composer, hal_graphics_composer_hwservice)`
      - rules for hwservice manager registry and discovery
    - `binder_call(hal_graphics_composer_client, servicemanager)`
      - client can call to service manager
    - `binder_call(hal_graphics_composer_server, servicemanager)`
      - server can call to service manager
    - `hal_attribute_service(hal_graphics_composer, hal_graphics_composer_service)`
      - rules for service manager registry and discovery
