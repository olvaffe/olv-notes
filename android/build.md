Android Build System
====================

## Get Started

- steps
  - `source build/envsetup.sh`
  - `lunch foo-trunk_staging-eng`
  - `m`
- other steps
  - `m installclean` removes all partition directories and images under
    `out/target/product/foo`
    - followed by `m`, it makes sure no stale files are in the final images
  - `m dist` generates `foo-img.zip` for fastboot update, and `foo-ota.zip`
    for ota, and foo-specific images
    - it generates all files added by `dist-for-goals`
- `source build/envsetup.sh`
  - `validate_current_shell` makes sure the shell is bash or zsh
  - `set_global_paths` updates `$PATH`
    - `build/soong/bin`
      - `hmm` shows help
      - `m` builds from the top of tree
      - `mm` builds from the current dir
      - `mmm` builds from the specified dir
      - these used to be defined in `build/envsetup.sh`
    - `build/bazel/bin`
    - `development/scripts`
    - `prebuilts/devtools/tools`
    - `prebuilts/misc/linux-x86/dtc`
    - `prebuilts/misc/linux-x86/libufdt`
    - `prebuilts/android-emulator/linux-x86_64`
  - `source_vendorsetup` sources `vendorsetup.sh` found under `device/`,
    `vendor/`, or `product/`
  - `addcompletions` adds auto-completions
  - the script also defines a bunch of useful functions
    - `gettop` prints the top dir
    - `croot` chdirs to the top dir
    - `lunch` selects a product/release/variant
- `lunch foo-trunk_staging-eng`
  - on V(?), the arg must be `<product>-<release>-<variant>`
    - `TARGET_PRODUCT=<product>`, such as `foo`
    - `TARGET_RELEASE=<release>`
      - `next` is the next release, and is the most stable one
      - `trunk` includes everything after the next release
      - `trunk_staging` includes staging changes as well, and is the least
        stable one
    - `TARGET_BUILD_VARIANT=<variant>`, such as `userdebug`
  - if no product specified, `print_lunch_menu` prints all products
    - it invokes `build/soong/soong_ui.bash --dumpvar-mode COMMON_LUNCH_CHOICES`
    - `COMMON_LUNCH_CHOICES` is defined by various `AndroidProducts.mk`
  - `build_build_var_cache` invokes
    `build/soong/soong_ui.bash --dumpvars-mode` with `TARGET_PRODUCT` set
    - this parses `device/*/$TARGET_PRODUCT.mk`
  - `set_stuff_for_environment` sets envvars
- `m` invokes `build/soong/soong_ui.bash --build-mode --all-modules` which
  invokes `runMake`

## Artifacts

- artifacts in `out`
- `combined-foo.ninja` is the ninja file
  - `build-foo.ninja` is generated by kati from Makefiles
  - `build-foo-package.ninja` is also generated by kati from Makefiles
  - `soong/build.foo.ninja` is generated by soong from `Android.bp`
- `host` is for host
  - `common` is java-based host tools
  - `linux-x86` is linux-x86 host tools
- `target` is for target
  - `common` is java-based target binaries
  - `product/foo` is foo-specific target binaries
    - `boot.img` consists of kernel, dtb, cmdline, etc.
    - `init_boot.img` consists the initrd
    - `product` and `product.img` are the product partition
    - `ramdisk` and `ramdisk.img` are the initrd
    - `root` is the root partition
    - `super.img` consists of
      - `product.img`
      - `system.img`
      - `system_dlkm.img`
      - `system_ext.img`
      - `vendor.img`
      - `vendor_dlkm.img`
    - `system` and `system.img` are the system partition
    - `system_dlkm` and `system_dlkm.img` are the system dlkm partition
    - `system_ext` and `system_ext.img` are the system ext partition
    - `userdata.img` is the data partition (and is empty)
    - `vbmeta.img` is the verified boot meta partition
    - `vendor` and `vendor.img` are the vendor partition
    - `vendor_dlkm` and `vendor_dlkm.img` are the vendor dlkm partition
- `verbose.log.gz` is the verbose log

## Soong

- soong `preProductConfigSetup` calls `FindSources` to generate various
  lists under `out/.module_paths/`
  - `Android.mk.list` for all first `Android.mk`
  - `CleanSpec.mk.list` for all first `CleanSpec.mk`
  - `AndroidProducts.mk.list` for all `AndroidProducts.mk` under `device/`,
    `vendor/`, or `product/`
  - `OWNERS.list` for all `OWNERS`
  - `METADATA.list` for all `METADATA`
  - `TEST_MAPPING.list` for all `TEST_MAPPING`
  - `Android.bp.list` for all `Android.bp`
  - `configuration.list` for all `*.mk` not matched above
- `./build/soong/soong_ui.bash --dumpvar-mode COMMON_LUNCH_CHOICES`
  - it is dispatched to `dumpVar`
  - `dumpVar` calls `DumpMakeVars`
  - `DumpMakeVars` calls `dumpMakeVars` to invoke
    `ckati -f build/make/core/config.mk dump-many-vars`
    - `ckati` is a `make` clone capable of converting makefiles to ninja files
- `./build/soong/soong_ui.bash --build-mode --all-modules --dir=$TOP`
  - it is dispatched to `runMake`
  - `runMake` calls `Build`
  - `runMakeProductConfig` calls `dumpMakeVars`
  - `runSoong` converts Android.bp to ninja
  - `runKatiBuild` converts Android.mk to ninja
    - `ckati -f build/make/core/main.mk`
  - `runKatiPackage` converts Android.mk to ninja
    - `ckati -f build/make/packaging/main.mk`
  - `runNinjaForBuild` runs ninja

## Make

- soong `--dumpvar-mode` uses `build/make/core/config.mk`
  - `include $(BUILD_SYSTEM)/envsetup.mk`
    - `include $(BUILD_SYSTEM)/product_config.mk`
      - `android_products_makefiles` is set to the contents of
        `out/.module_paths/AndroidProducts.mk.list` plus
        `$(SRC_TARGET_DIR)/product/AndroidProducts.mk`
        - `$(SRC_TARGET_DIR)` expands to `build/make/target`
      - `_read-ap-file` is called to read all `AndroidProducts.mk`
      - `$(call import-products, $(current_product_makefile))` imports the
        current product's makefile (set by `PRODUCT_MAKEFILES` in
        `AndroidProducts.mk`)
      - `TARGET_DEVICE := $(PRODUCT_DEVICE)`
    - `include $(BUILD_SYSTEM)/board_config.mk`
      - `board_config_mk` is set to `*/$(TARGET_DEVICE)/BoardConfig.mk` and is
        included
  - `include $(BUILD_SYSTEM)/dumpvar.mk`
    - `dumpvar.mk` defines `dump-many-vars` target to dump make variables
- soong `--build-mode` uses `build/make/core/main.mk` (and more)
  - `DEFAULT_GOAL := droid`

## Device Makefiles

- <https://source.android.com/docs/setup/create/new-device>
- `device/company/board/AndroidProducts.mk` defines the products
  - `PRODUCT_MAKEFILES` specifies the product makefile(s), such as
    `$(LOCAL_DIR)/foo.mk`
  - `COMMON_LUNCH_CHOICES` specifies the lunch choices, such as
    `foo-userdebug`
- `device/foo/BoardConfig.mk` defines the board config
  - ideally, only `BOARD_*` and `TARGET_*` variables are set
- `device/company/board/foo.mk` defines the product config
  - it uses `(call inherit-product, ...)` to inherit other products
  - it often inherits generic products under `build/make/target/product`
    - `core_64_bit.mk` enables 64-bit support
    - `generic_ramdisk.mk` enables generic ramdisk support
    - `generic_no_telephony.mk` enables generic tablet support, without
      telephony
    - `core_minimal.mk` enables minimal product
  - varous `PRODUCT_*` variables are set
    - `PRODUCT_USE_DYNAMIC_PARTITIONS := true`

## Installed Targets

- `git grep ^INSTALLED` under `build/make/`
- `core/Makefile`
  - <https://source.android.com/docs/core/architecture/partitions/generic-boot>
    - `INSTALLED_RECOVERYIMAGE_TARGET := $(PRODUCT_OUT)/recovery.img`
      - this is no longer needed with A/B boot
    - `INSTALLED_RAMDISK_TARGET := $(PRODUCT_OUT)/ramdisk.img`
      - it packages from `$(TARGET_RAMDISK_OUT)`
    - `INSTALLED_BOOTIMAGE_TARGET := $(PRODUCT_OUT)/boot.img`
      - kernel packaged by `mkbootimg`
    - `INSTALLED_INIT_BOOT_IMAGE_TARGET := $(PRODUCT_OUT)/init_boot.img`
      - ramdisk packaged by `mkbootimg`
    - `INSTALLED_VENDOR_RAMDISK_TARGET := $(PRODUCT_OUT)/vendor_ramdisk.img`
      - vendor ramdisk packaged by `mkbootimg`
    - `INSTALLED_VENDOR_BOOTIMAGE_TARGET := $(PRODUCT_OUT)/vendor_boot.img`
  - `INSTALLED_RECOVERY_BUILD_PROP_TARGET := $(TARGET_RECOVERY_ROOT_OUT)/prop.default`
  - `INSTALLED_SYSTEMIMAGE_TARGET := $(PRODUCT_OUT)/system.img`
  - `INSTALLED_USERDATAIMAGE_TARGET := $(PRODUCT_OUT)/userdata.img`
  - `INSTALLED_VENDORIMAGE_TARGET := $(PRODUCT_OUT)/vendor.img`
  - `INSTALLED_PRODUCTIMAGE_TARGET := $(PRODUCT_OUT)/product.img`
  - `INSTALLED_SYSTEM_EXTIMAGE_TARGET := $(PRODUCT_OUT)/system_ext.img`
  - `INSTALLED_SYSTEM_DLKMIMAGE_TARGET := $(PRODUCT_OUT)/system_dlkm.img`
  - `INSTALLED_VBMETAIMAGE_TARGET := $(PRODUCT_OUT)/vbmeta.img`
  - `INSTALLED_FASTBOOT_INFO_TARGET := $(PRODUCT_OUT)/fastboot-info.txt`
  - `INSTALLED_MISC_INFO_TARGET := $(PRODUCT_OUT)/misc_info.txt`
  - `INSTALLED_SUPERIMAGE_TARGET := $(PRODUCT_OUT)/super.img`
    - <https://source.android.com/docs/core/ota/dynamic_partitions/implement>
  - `INSTALLED_SUPERIMAGE_EMPTY_TARGET := $(PRODUCT_OUT)/super_empty.img`
- `core/definitions.mk`
  - `INSTALLED_RADIOIMAGE_TARGET` is populated by `add-radio-file`
- `core/sysprop.mk`
  - `INSTALLED_BUILD_PROP_TARGET := $(TARGET_OUT)/build.prop`
    - `generate-common-build-props` generates `ro.product.system.*` and
      `ro.system.build.*`
    - `buildinfo.py` generates `ro.build.*`
    - `ADDITIONAL_SYSTEM_PROPERTIES`
    - `PRODUCT_SYSTEM_PROPERTIES`
    - `PRODUCT_SYSTEM_DEFAULT_PROPERTIES`
  - `INSTALLED_VENDOR_BUILD_PROP_TARGET := $(TARGET_OUT_VENDOR)/build.prop`
    - `generate-common-build-props` generates `ro.product.vendor.*` and
      `ro.vendor.build.*`
    - `ADDITIONAL_VENDOR_PROPERTIES`
      - `ro.hwui.use_vulkan` is set to `$(TARGET_USES_VULKAN)` and decides
        whether hwui uses skiavk or skiagl
    - `PRODUCT_VENDOR_PROPERTIES`
      - `ro.hardware.hwcomposer` determines what `hw_get_module` dlopens
      - `ro.hardware.egl` determines what libEGL dlopens
      - `ro.hardware.vulkan` determines what libvulkan dlopens
      - `debug.renderengine.backend` determines what SF render
        engine uses
    - `PRODUCT_DEFAULT_PROPERTY_OVERRIDES`
    - `PRODUCT_PROPERTY_OVERRIDES`
      - `ro.hardware.gralloc` determines what `hw_get_module` dlopens
  - `INSTALLED_PRODUCT_BUILD_PROP_TARGET := $(TARGET_OUT_PRODUCT)/etc/build.prop`
    - `generate-common-build-props` generates `ro.product.product.*` and
      `ro.product.build.*`
    - `ADDITIONAL_PRODUCT_PROPERTIES`
    - `PRODUCT_PRODUCT_PROPERTIES`
  - `INSTALLED_ODM_BUILD_PROP_TARGET := $(TARGET_OUT_ODM)/etc/build.prop`
    - `generate-common-build-props` generates `ro.product.odm.*` and
      `ro.odm.build.*`
    - `ADDITIONAL_ODM_PROPERTIES`
    - `PRODUCT_ODM_PROPERTIES`
  - `INSTALLED_VENDOR_DLKM_BUILD_PROP_TARGET := $(TARGET_OUT_VENDOR_DLKM)/etc/build.prop`
    - `generate-common-build-props` generates `ro.product.vendor_dlkm.*` and
      `ro.vendor_dlkm.build.*`
  - `INSTALLED_ODM_DLKM_BUILD_PROP_TARGET := $(TARGET_OUT_ODM_DLKM)/etc/build.prop`
    - `generate-common-build-props` generates `ro.product.odm_dlkm.*` and
      `ro.odm_dlkm.build.*`
  - `INSTALLED_SYSTEM_DLKM_BUILD_PROP_TARGET := $(TARGET_OUT_SYSTEM_DLKM)/etc/build.prop`
    - `generate-common-build-props` generates `ro.product.system_dlkm.*` and
      `ro.system_dlkm.build.*`
  - `INSTALLED_SYSTEM_EXT_BUILD_PROP_TARGET := $(TARGET_OUT_SYSTEM_EXT)/etc/build.prop`
    - `generate-common-build-props` generates `ro.product.system_ext.*` and
      `ro.system_ext.build.*`
    - `PRODUCT_SYSTEM_EXT_PROPERTIES`
  - `INSTALLED_RAMDISK_BUILD_PROP_TARGET := $(TARGET_RAMDISK_OUT)/system/etc/ramdisk/build.prop`
    - `generate-common-build-props` generates `ro.product.bootimage.*` and
      `ro.bootimage.build.*`
- `core/tasks/oem_image.mk`
  - `INSTALLED_OEMIMAGE_TARGET := $(PRODUCT_OUT)/oem.img`
    - it packages from `$(TARGET_OUT_OEM)`
- `target/board/Android.mk`
  - `INSTALLED_ANDROID_INFO_TXT_TARGET := $(PRODUCT_OUT)/android-info.txt`
    - e.g., it can be generated from `TARGET_BOARD_INFO_FILES`

## Desktop

- if `PACK_DESKTOP_FILESYSTEM_IMAGES`, it builds `PACK_IMAGE_TARGET`
  (`android-desktop_image.bin`)
  - this is a live usb image
  - `PACK_IMAGE_SCRIPT` is `pack_image` script
- it depends on `IMAGES` which is
  - `INSTALLED_BOOTIMAGE_TARGET` (`boot.img`)
  - `INSTALLED_SUPERIMAGE_TARGET` (`super.img`)
  - `INSTALLED_INIT_BOOT_IMAGE_TARGET` (`init_boot.img`)
  - `INSTALLED_VENDOR_BOOTIMAGE_TARGET` (`vendor_boot.img`)
  - `INSTALLED_VBMETAIMAGE_TARGET` (`vbmeta.img`)
  - `INSTALLED_USERDATAIMAGE_TARGET` (`userdata.img`)
- the disk layout is similar to
  <https://chromium.googlesource.com/chromiumos/platform/crosutils/+/refs/heads/main/build_library/disk_layout_v3.json>
  - see also <https://source.android.com/docs/core/architecture/partitions>
  - p1: `userdata`, 4G
  - p2: `KERN-A`, 32M, only by recovery usb image
  - p3: `super`, 8G, `super.img`
    - `m superimage` runs `build_super_image` which runs `lpmake` to pack
      these to `super.img`
      - `system.img`
      - `system_ext.img`
      - `system_dlkm.img`
      - `vendor.img`
      - `vendor_dlkm.img`
      - `product.img`
  - p4: `KERN-B`, unused
  - p5: `ROOT-B`, unused
  - p6: `KERN-C`, unused
  - p7: `ROOT-C`, unused
  - p8: `OEM`, 4M
  - p9: `MINIOS-A`, 128M
  - p10: `MINIOS-B`, 128M
  - p11
  - p12: `EFI-SYSTEM`, 64M, only by non-chromebook uefi system
  - p13: `boot_a`, 64M, `boot.img`
    - `m bootimage` runs `mkbootimg` to pack `INSTALLED_KERNEL_TARGET`
      (`kernel`) to `boot.img`
  - p14: `boot_b`, 64M
  - p15: `vbmeta_a`, 4M, `vbmeta.img`
    - `m vbmetaimage` runs `avbtool make_vbmeta_image` to generate
      `vbmeta.img`
  - p16: `vbmeta_b`, 4M
  - p17: `metadata`, 16M, generated
  - p18: `init_boot_a`, 32M, `init_boot.img`
    - `m initbootimage` runs `mkbootimg` to pack `INSTALLED_RAMDISK_TARGET`
      (`ramdisk.img`) to `init_boot.img`
  - p19: `init_boot_b`, 32M
  - p20: `vendor_boot_a`, 32M, `vendor_boot.img`
    - `m vendorbootimage` runs `mkbootimg` to pack these to `vendor_boot.img`
      - `INTERNAL_KERNEL_CMDLINE` (`BOARD_KERNEL_CMDLINE`)
      - `INTERNAL_VENDOR_RAMDISK_TARGET` (`vendor_ramdisk.cpio.lz4`)
      - `INTERNAL_VENDOR_RAMDISK_FRAGMENTS` (`recovery.cpio.lz4`)
  - p21: `vendor_boot_b`, 32M
  - p22: `pvmfw_a`, 4M, `pvmfw.img`
  - p23: `pvmfw_b`, 4M
  - p24: `misc`, 4M
  - p25: `RWFW-A`, unused
  - p26: `RWFW-B`, unused

## boot.img (of x86): 

- generated by `external/genext2fs/mkbootimg_ext2.sh`
- contains:

        /cmdline <- BOARD_KERNEL_CMDLINE
        /kernel <- INSTALLED_KERNEL_TARGET
        /ramdisk <- INSTALLED_RAMDISK_TARGET
        /boot/grub/menu.list (optional)
- ramdisk collects files under `TARGET_ROOT_OUT = $(PRODUCT_OUT)/root`

## system.img (of x86):

- collects files under `TARGET_OUT = $(PRODUCT_OUT)/system`

## disk installer*.img:

- `make installer_img (bootable/diskinstaller/config.mk)`
- installer.img contains:

        [MBA]: grub.bin
        [first partition]: installer_tmp.img
        [second partition]: installer_data.img
- `installer_tmp.img` is like boot.img, contains:

        /kernel <- INSTALLED_KERNEL_TARGET
        /ramdisk
- ramdisk contains, among others:

        /init.rc <- from diskinstaller
        /etc/disk_layout.conf <- from eee_701
        /etc/installer.conf <- from diskinstaller
        /bin/installer <- diskinstaller
- installer_data.img contains:

        /bootldr.bin <- grub.bin
        /boot.img <- INSTALLED_BOOTIMAGE_TARGET
        /system.img <- INSTALLED_SYSTEMIMAGE
        /userdata.img <- INSTALLED_USERDATAIMAGE_TARGET

## qemu-x86 android

- based on eeepc_701
- CAUTIONS
  - make sure external/e2fsprogs is not commented out when make installer_img
  - make sure phone policy is not preloaded
- cmdline: noapic, module: ne2k-pci
- steps
  - qemu-img create mydroid.raw 2G
  - dd if=grub/grub.bin of=mydroid.raw conv=notrunc
  - qemu -hda mydroid.raw -hdb installer.img
    and install android
    In grub choose fist boot option and use second harddisk
  - qemu -hda mydroid.raw -redir tcp:5555::5555 to run
- cupcake fixes:
  `in eeepc_701/init.rc, setprop ro.HOME_APP_ADJ 4 && setprop ro.HOME_APP_MEM 4096`
- disk layout see eee_701/disk-layout.conf

## eeepc android

- just like on qemu
- in `init.eee_701.sh`, replace dhcp by `ifconfig eth0 192.168.0.2.2 up`
- in `BoardConfig.mk`, replace `vga=788` by `video=i915 i915.modeset=1`
- patch inteldrmfb to use 16bits fb
- `ADBHOST=192.168.0.202 ./adb shell`
- `mkdir /data/a` and `mount -t ext2 -o rw /dev/block/sda3 /data/a` to modify
  cmdline
- `mount -t ext3 -o remount,rw /dev/block/sda6 /system` and modify
  init.eee_701.sh

---

## Layers

- The top-level description is a product
  - e.g. myProduct, myProduct_zh-tw, myProduct_with_camera, myProduct_sdk
- a device is used on one or more products
  - e.g. an SoC plus peripherals
- a board is used on one or more devices
  - e.g. an SoC
- an arch is used on one or more boards
  - cpu architecture

## A new product

- `mkdir -p vendor/myCompany/products/`
- `vi vendor/myCompany/products/AndroidProducts.mk`
  - `PRODUCT_MAKEFILES := $(LOCAL_DIR)/myProductA.mk`
- `vi vendor/myCompany/products/myProductA.mk`
  - `$(call inherit-product, $(SRC_TARGET_DIR)/product/generic.mk)`
  - `PRODUCT_NAME := <name>`
  - `PRODUCT_DEVICE := <device_name>`
- `mkdir -p vendor/myCompany/<device_name>`
- `vi vendor/myCompany/<device_name>/BoardConfig.mk`
  - `TARGET_NO_BOOTLOADER := true`
  - `TARGET_USE_GENERIC_AUDIO := true`
- `build/core/config.mk`
  - includes `envsetup.mk` which includes `board_config.mk`
    - `find -name AndroidProducts.mk` in `device`, `vendor`, and
      `build/target/product/AndroidProducts.mk`
    - includes all files defined by `PRODUCT_MAKEFILES`
    - especially, `TARGET_DEVICE` is determined
  - includes `BoardConfig.mk`s
    - `build/target/board/$(TARGET_DEVICE)/BoardConfig.mk`
    - `device/*/$(TARGET_DEVICE)/BoardConfig.mk`
    - `vendor/*/$(TARGET_DEVICE)/BoardConfig.mk`
  - later when `target/board/Android.mk` is included, it
    - includes `$(TARGET_DEVICE_DIR)/AndroidBoard.mk`, where `TARGET_DEVICE_DIR`
      is the directory holding `BoardConfig.mk`

## BUILD system:

- build/core/build-system.html
- <http://source.android.com/porting/build_system.html>
- make anything with showcommands to show commands
- Installs modules tagged with: eng, debug, user, and/or development.
- Installs non-APK modules that have no tags specified.
- Installs APKs according to the product definition files, in addition to tagged APKs.
- ro.secure=0
- ro.debuggable=1
- ro.kernel.android.checkjni=1
- adb is enabled by default. 
- binary.mk: take `LOCAL_SRC_FILES`, generate rules for `c_normal_objects := $(addprefix $(intermediates)/,$(c_normal_sources:.c=.o))`
- modules are built to `LOCAL_BUILT_MODULE`, and copied to `LOCAL_INSTALLED_MODULE`

## Product

- `main.mk -> config.mk -> envsetup.mk -> product_config.mk`
- All `AndroidProducts.mk` are checked and the collections of all
  `PRODUCT_MAKEFILES` are sourced
- `PRODUCT_MAKEFILES -> PRODUCT_NAME, PRODUCT_DEVICE, etc.`
- then we get TARGET_DEVICE from `PRODUCT_DEVICE`
- in config.mk `TARGET_DEVICE -> board_config_mk = build/target/board/$(TARGET_DEVICE)/BoardConfig.mk vendor/*/$(TARGET_DEVICE)/BoardConfig.mk`
- later build/target/board/Android.mk is included -> vendor/asus/eee_701/AndroidBoard.mk

## build/core/java_library.mk using services.jar as an example

- frameworks/base/services/jni/: `BUILD_SHARED_LIBRARY` libandroid_servers.so
- frameworks/base/services/java/: `BUILD_JAVA_LIBRARY` services.jar
- Then

        BUILD_JAVA_LIBRARY is set to build/core/java_library.mk
        LOCAL_JAVA_LIBRARIES := core ext framework $(LOCAL_JAVA_LIBRARIES)
        full_classes_jar := $(intermediates.COMMON)/classes.jar
        built_dex := $(intermediates.COMMON)/classes.dex

  Sources are transform-java-to-classes.jar to classes-full-debug.jar first, then copied to classes.jar
  classes.jar is transform-classes.jar-to-dex to classes.dex
  If static, classes.jar is copied to javalib.jar.  Otherwise, classes.dex and java_resource_sources is
    add-dex-to-package and add-java-resources-to-package to javalib.jar.
- `LOCAL_JAVA_LIBRARIES`: expanded to jar files by java-lib-files and stored in `full_java_libs`
- `all_java_sources`: list sources, which come from:

        LOCAL_SRC_FILES
        LOCAL_INTERMEDIATE_SOURCES additional package specified sources, under OUT_COMMON_INTERMEDIATES
        aidl_java_sources aidl generated sources found under $(intermediates.COMMON)/src/
- transform-java-to-classes.jar

        echo "target Java: blah"
        Create $(intermediates)/classes/
        Unzip static libraries' jar to classes/
        all_java_sources is dumped to classes/java-source-list
        sources under src/ is also dumped (for aapt generated files)
        javac to compile
        classes/java-source-list is removed and classes/ is jar'ed.
- .dex and resource files packages into javalib.jar, which is then copied to services.jar

## framework.jar

- framework.jar built by frameworks/base/Android.mk
- `LOCAL_JAVA_RESOURCE_FILES += $(LOCAL_PATH)/preloaded-classes`
   to add preloaded-classes to framework.jar
- sources generated by framework-res are specified in `LOCAL_INTERMEDIATE_SOURCES`
  and are also compiled in

## build/core/package.mk

- If there is res, `R_file_stamp` is defined
- package.apk depends on `R_file_stamp`, which:

        Calls create-resource-java-files
        Copy generated Manifest.java and R.java to $(TARGET_COMMON_OUT_ROOT)/R/
        Use last R.java as R.stamp
- create-resource-java-files: 

        PRIVATE_ANDROID_MANIFEST is $(LOCAL_PATH)/AndroidManifest.xml
        PRIVATE_RESOURCE_DIR is $(LOCAL_PATH)/res
        PRIVATE_ASSET_DIR is $(LOCAL_PATH)/assets
        PRIVATE_SOURCE_INTERMEDIATES_DIR is under $(intermediates.COMMON)/src/
        PRIVATE_RESOURCE_PUBLICS_OUTPUT is $(intermediates.COMMON)/public_resources.xml
        PRIVATE_AAPT_INCLUDES is $(framework_res_package_export) unless building framework-res.apk
  It generates public_resources.xml and src/{android,com/android/PACKAGE}/{Manifest,R}.java
- After R (resource java files) are generated, compile!  See transform-java-to-classes.jar.
- package.apk is aapt generated by combining assets, res, and classes.dex.

## framework-res.apk

- built by frameworks/base/core/res/Android.mk
- a special package without any source
- framework-res generates android/R.java and com/android/internal/R.java
- framework-res has no source and does not compile any source
  The generated sources are used by framework.jar
- `LOCAL_EXPORT_PACKAGE_RESOURCES` set to true to generate package-export.apk in $(intermediates.COMMON)

## android.policy.jar

- one of the policies is compiled and copy to android.policy.jar

## Slowness: Analysis of `main.mk`

- `findleaves.sh` takes 0.5s
- `include $(subdir_makefiles)` takes 10s to include all `Android.mk`
- `include $(BULD_SYSTEM)/Makefile` is fast but gives
  `/bin/bash: Argument list too long` because it includes all `tasks/localize.mk`
- `mm`'s slowness comes from `findleaves.sh`

## Modules to install

- user tag
  - `GRANDFATHERED_USER_MODULES`
    - a bunch of host tools (for building the image)
    - all system tools: app_process, bootanimation, netcfg, linker, ...
    - all daemons: adbd, surfaceflinger, mediaserver, installd, vold...
    - config files
    - Android runtime: a bunch of JAR's
    - shared libraries including default EGL/GLES, gralloc
  - `PRODUCT_PACKAGES`: apps, libexpat, dalvikvm, libdvm, and etc.
- eng tag
  - Development.apk, Term.apk, and etc.
  - command line tools
- debug tag
  - emulator/emulator-arm/emulator-x86/qemu/android-arm/qemu-android-x86
  - gdbserver, strace, and many other command tools
- default
  - defined by `ALL_DEFAULT_INSTALLED_MODULES`
  - consists of toolbox links
- `ALL_DEFAULT_INSTALLED_MODULES` is then replaced by the list above and
  `core/Makefile` is included
- A module marked as optional is not to be installed (to dst directory)
  - but it is still built (in `obj/` directory) because
    `LOCAL_DONT_CHECK_MODULE` is not set.

## building a module

- `ALL_MODULES` is the names of all modules
- `LOCAL_MODULE` is `foo`
  - `LOCAL_BUILT_MODULE` is the full path to the module in `obj`
  - `LOCAL_INSTALLED_MODULE` is the full path to the module in the dst
    directory
  - `LOCAL_MODULE` depends on `LOCAL_BUILT_MODULE` and `LOCAL_INSTALLED_MODULE`
  - `LOCAL_INSTALLED_MODULE` depends on `LOCAL_BUILT_MODULE`
  - for shard libraries,
    - `LOCAL_BUILT_MODULE` is `obj/lib/xxxx.so`
    - it depends on `obj/<CLASS>/<MODULE>/LINKED/xxxx.so`
    - which in turn depends on the object files
  - when `LOCAL_SHARED_LIBRARIES` or `LOCAL_STATIC_LIBRARIES` is listed,
    - `LOCAL_BUILT_MODULE` depends on the `LOCAL_BUILT_MODULE` of libraries
    - `LOCAL_INSTALLED_MODULE` depends on the `LOCAL_INSTALLED_MODULE` of shared
      libraries

## SDK

- SDK depends on a normal build of the emulator and images
  - It could be a `full-eng` build or `sdk-eng` build
  - The latter is more or less a subset of the former
  - There are one or two places need to be fixed if building SDK with `full-eng`
- `sdk` target
- `win_sdk` target
  - to add new files to the SDK
    - Find `SDK_ONLY` in `build/core/main.mk`.  Add more directories to the
      search pathes.
    - Go to `development/build/tools`.  Edit `windows_sdk.mk` to add new modules
      to be cross-compiled.  Edit `patch_windows_sdk.sh` to copy them to the
      right places in the SDK.
