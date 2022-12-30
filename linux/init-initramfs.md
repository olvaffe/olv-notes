initramfs
=========

## Kernel

- during early init (`start_kernel`), `init_mount_tree` mounts `rootfs` and
  makes it pwd and /
  - it is called from `mnt_init` from `vfs_caches_init`
  - `rootfs` is a `tmpfs`
- when kernel is fully initialized (`rest_init`), it spawns the first thread
  with pid 1 to execute `kernel_init`
- pid 1 calls `populate_rootfs` or `default_rootfs` to populate rootfs
  - called from `do_initcalls` from `do_basic_setup` from
    `kernel_init_freeable`
  - `default_rootfs` is called when `CONFIG_BLK_DEV_INITRD` is not set.  It
    creates `/dev/console` and `/root` in rootfs
  - otherwise, `populate_rootfs` is called to unpack cpio.  When no cpio is
    supplied at build time or by bootloader, it uses `usr/default_cpio_list`
    to create `/dev/console` and `/root` in rootfs
- pid 1 calls `console_on_rootfs` to open `/dev/console` and dup it to fd 0,
  1, and 2
  - when initramfs cpio does not have the node, this is skipped
- pid 1 checks `/init`.  If the unpacked cpio has it, pid 1 `execve`s and
  handles control to userspace
- if there is no `/init`, kernel calls `prepare_namespace` to mount `root=`
  - `name_to_dev_t` parses `root=` to get root device major:minor and save it
    to `ROOT_DEV`
  - `mount_root` creates `/dev/root` pointing to `ROOT_DEV` and calls
    `mount_block_root` to mount it to `/root` and `chdir` to it
    - on success, dmesg prints `VFS: Mounted root (xxx filesystem) readonly on device xxx:xxx.`
    - on failure, dmesg prints some info and `VFS: Unable to mount root fs on xxx`
  - `devtmpfs_mount` mounts devtmpfs to `/root/dev`
  - `init_mount` move-mounts `/root` to `/`
  - `init_chroot` chroots
- pid 1 finally `execve`s `/sbin/init` (when there is no `/init` in initramfs)

## initramfs

- the kernel initramfs unpacker checks if the cpio starts with "070701", that
  is, the new portable format
- kernel thread of pid 1 `execve`s initramfs's `/init` (or `rdinit=`) with
  - `argv = { "init", NULL }`
  - `envp = { "HOME=/", "TERM=linux", NULL }`
- a minimal busybox-based initramfs can do these in `/init` to get a shell
  - `#!/bin/busybox sh`
  - `mkdir /proc; mount -t proc none /proc`
  - `mkdir /sys; mount -t sysfs none /sys`
  - `mkdir /dev; mount -t devtmpfs none /dev`
  - `exec >/dev/console 2>/dev/console </dev/console`
  - `exec sh -i`
  - if busybox relies on symlinks or `$PATH` to find its applets, prepend
    - `/bin/busybox --install -s /bin`
    - `export PATH=/bin`
- and to create the cpio archive
  - `find . | cpio -o -H newc -R root:root > ../initramfs.cpio`
