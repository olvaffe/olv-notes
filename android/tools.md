avd
* ./android create avd --target 2 --name olv

qemu
* http://developer.android.com/guide/developing/tools/emulator.html
* CPU: ARM926EJ-S [41069265] revision 5 (ARMv5TEJ), cr=00093177
* PageUp for menu key
* ./emulator -system arm-by-pinky/system.img -data arm-by-pinky/userdata.img \
  -ramdisk arm-by-pinky/ramdisk.img -kernel arm-by-pinky/zImage
* -show-kernel is useful
* -logcat '*:v'

kernel
* adb pull /proc/config.gz . to get current kernel config
* +CONFIG_STAGING=y
  +CONFIG_ANDROID=y
  +CONFIG_ANDROID_BINDER_IPC=y
  +CONFIG_ANDROID_LOGGER=y
  +CONFIG_ANDROID_TIMED_OUTPUT=y
  +CONFIG_ANDROID_LOW_MEMORY_KILLER=y
  -CONFIG_ANDROID_PMEM
* make ARCH=arm CROSS_COMPILE=../prebuilt/linux-x86/toolchain/arm-eabi-4.3.1/bin/arm-eabi-

adb
* adb remount is useful
* adb kill-server

hierarchyviewer
* useful UI debuggin tool

## adb

* adb can be run on host and device
  * it is called `adbd` when on the device
* It calls `usb_init`
  * on device, it opens `/dev/android_adb`, which waits for connection.
  * on host, it scans `/dev/bus/usb` for usb devices supporting adb
    * the device might be both readable and writable.
* It calls `local_init`
  * on device, it listens on 5555 port.
  * on host, it connects to 5555 port of `ADBHOST`.
