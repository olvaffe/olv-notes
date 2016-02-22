with CONFIG_BLK_DEV_INITRD

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
