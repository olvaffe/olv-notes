Chrome OS AP firmware
=====================

## AP firmware (BIOS)

- `emerge-<board> chromeos-bootimage`
  - depends on `coreboot`
    - `/build/$BOARD/firmware/$DEVICE/coreboot.rom`
    - a CBFS archive
    - no payload
  - depends on `depthcharge`
    - `/build/$BOARD/firmware/$BOARD/depthcharge/depthcharge.elf`
    - a coreboot payload
- it generates
  - `/build/$BOARD/firmware/image-$DEVICE.bin`
  - `/build/$BOARD/firmware/image-$DEVICE.serial.bin` for debug
  - others

## Update BIOS

- `emerge-<board> chromeos-bootimage`
- `futility update -i image-<board>.bin` on DUT
- `futility update --servo -i image-<board>.bin` on host

## `cbfstool`

- the bootimage is a CBFS archive
  - repacked from `coreboot.rom` which is also a CBFS archive
- to see the contents,
  - `cbfstool image-<board>.bin layout`
  - `cbfstool image-<board>.bin print -r COREBOOT`
  - `cbfstool image-<board>.bin print -r RW_LEGACY`

## Depthcharge

- depthcharge is a payload of coreboot
- it links to `libpayload`
- it links to vboot
  - `vboot_reference`

## altfw

- to use altfw on DUT,
  - `crossystem dev_boot_legacy=1`
  - `Ctrl-L` in boot prompt
- when building chromeos-bootimage,
  - `USE=seabios` to include seabios
  - `USE=tianocore` to include UEFI
  - `USE=u-boot` to include u-boot
- `cbfstool /build/$BOARD/firmware/image-$DEVICE.bin print -r RW_LEGACY` to
  see the included payloads
- <https://chromium.googlesource.com/chromiumos/docs/+/refs/heads/master/developer_mode.md#alt-firmware>
  - `flashrom -r /tmp/bios.bin`
  - `cbfstool /tmp/bios.bin print -r RW_LEGACY`
  - `cbfstool /tmp/bios.bin add-payload -r RW_LEGACY -c lzma -n <your bootloader name> -f <path/to/your/bootloader.elf>`
  - `cbfstool /tmp/bios.bin extract -r RW_LEGACY -n altfw/list -f /tmp/altfw.txt`
  - `cbfstool /tmp/bios.bin remove -r RW_LEGACY -n altfw/list`
  - `cbfstool /tmp/bios.bin add -r RW_LEGACY -n altfw/list -f /tmp/altfw.txt -t raw`
  - `cbfstool /tmp/bios.bin remove -r RW_LEGACY -n cros_allow_auto_update`
  - `flashrom -w /tmp/bios.bin -i RW_LEGACY`
  - `crossystem dev_boot_legacy=1`
