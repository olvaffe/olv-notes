initramfs
=========

## cpio archive

- `CONFIG_BLK_DEV_INITRD` enables initramfs/initrd support
- `usr/initramfs_data.S` defines `__initramfs_size` in `.init.ramfs` section
  - when `CONFIG_INITRAMFS_SOURCE` is specified, it includes the specified
    cpio archive
  - otherwise, it generates a cpio archive according to
    `usr/default_cpio_list`
  - the cpio archive is embedded in the kernel image
- `INIT_RAM_FS` defines `__initramfs_start` and includes `.init.ramfs` section
  to help find the embedded archive

## cmdline

- `initrd=` requests the bootloader to load an external cpio archive
  - `initrd_start` and `initrd_end` are set to the address of the loaded
    cpio archive in memory
- `init=` is handled by `init_setup` and updates `execute_command`
  - default to `/sbin/init`
- `rdinit=` is handled by `rdinit_setup` and updates `ramdisk_execute_command`
  - default to `/init`
- `root=` is handled by `root_dev_setup` and updates `saved_root_name`

## mount

- `populate_rootfs` is called by `do_initcalls`
  - it unpacks the built-in initramfs at `__initramfs_start`
  - it then unpacks the external initramfs at `initrd_start`
  - if unpack of `initrd_start` failed and `CONFIG_BLK_DEV_RAM` is set, assume
    `initrd_start` points to a ramdisk image instead
- without `CONFIG_BLK_DEV_INITRD`, `default_rootfs` is called instead to
  populate a minimal rootfs

## init

- `kernel_init` kthread has pid 1 and runs `kernel_init`.  At the end, it
  `do_execve` init and become the userspace pid 1.  Before that...
- `console_on_rootfs` opens `/dev/console` and dups to fd 0, 1, and 2
- if initramfs contains `/init`,  `ramdisk_execute_command` is set to `/init`
  - otherwise, `prepare_namespace` sets `ROOT_DEV` according to `root=` and
    mounts the real rootfs
- `run_init_process` calls `do_execve`
