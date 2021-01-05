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
