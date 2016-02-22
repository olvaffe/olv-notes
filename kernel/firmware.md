Kernel and Firmware
===================

## `request_firmware`

* The one stop.
  * `Documentation/firmware_class/README`.
* e100 asks for `e100/d101m_ucode.bin`, and udev receives an event where it
  * `echo 1 > /sys/$DEVPATH/loading`
  * `cat $FIRMWARE > /sys/$DEVPATH/data`
  * `echo 0 > /sys/$DEVPATH/loading`
  and `request_firmware` returns.

## Firmware and KBuild

* It's easier to see an example, 9ac32e1bc0518b01b47dd34a733dce8634a38ed3
* `MODULE_FIRMWARE("firmware-name")` is added to the driver.
  * This is only informative.
* `fw-shipped-$(CONFIG_E100) += e100/d101m_ucode.bin` is added to
  `firmware/Makefile`.
* firmwares are put under `firmware/` in ihex or h16 forms
  * To convert ihex to bin, `objcopy -Iihex -Obinary`.
  * To convert ihex/h16 to fw, `ihex2fw`.
  * It is done automatically by the build system.
* If `CONFIG_FIRMWARE_IN_KERNEL` is defined,
  * firmware.bin is generated
  * firmware.gen.S is generated, which `.incbin` firmware.bin
  * It is compiled into firmware.gen.o.
  * All of firmware objects are compiled into the kernel.
    * They are in section `.builtin_fw` and are collected by
      `include/asm-generic/vmlinux.lds.h`
