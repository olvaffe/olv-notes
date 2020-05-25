initramfs
=========

## initramfs

- `CONFIG_BLK_DEV_INITRD` enables initramfs/initrd support
- `usr/initramfs_data.S` defines `__initramfs_size` in `.init.ramfs` section
  - when `CONFIG_INITRAMFS_SOURCE` is specified, it includes the specified
    cpio archive
  - otherwise, it generates a cpio archive according to
    `usr/default_cpio_list`
- `INIT_RAM_FS` defines `__initramfs_start` and includes `.init.ramfs` section
- cmdline `initrd=` requests the bootloader to load an external cpio archive
  - `initrd_start` and `initrd_end` are set to the address of the loaded
    cpio archive in memory



INIT_DATA_SECTION




* at build time, a cpio archive is put in .init.ramfs section, which is accessible through  __initramfs_start, __initramfs_end
* in arch, initrd_start and initrd_end is set (because of initrd=)
* rootfs_initcall populate_rootfs() prepares the rootfs
* if there is no '/init', prepare_namespace()
* finally in init_post(), '/dev/console' is opened, '/init' or '/sbin/init' is executed.
* or those specified by init= or rdinit=

populate_rootfs:
* unpack .init.ramfs
* if initrd_start, and if CONFIG_BLK_DEV_RAM, dump to /initrd.image if failed to unpack
* if no CONFIG_BLK_DEV_RAM, simply unpack initrd_start

prepare_namespace:
* ROOT_DEV (major:minor) is determined from root=
* try initrd_load, which dump /initrd.image to /dev/ram and mount it to execute /linuxrc
* if failed, mount root=
