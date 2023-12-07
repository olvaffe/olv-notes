Character Devices
=================

## Filesystem

- `mknod` creates an inode with `def_chr_fops` as the fops
- whenever the inode is opened, `chrdev_open` looks up the corresponding
  `cdev` from `cdev_map`.  `filp->f_op` is replaced by `cdev->ops` and the
  driver takes over from here.
- the driver should have called `cdev_add` to add a cdev to `cdev_map`
- data path is
  - userspace writes
  - driver's `file_operations`
- unlike block devices, whose data path is
  - userspace writes
  - page cache
  - bio
  - io request queue
  - io scheduler
  - driver translates io requests to native commands and submits

## Driver

- the driver calls `register_chrdev_region` to add a `char_device_struct` to
  `chrdevs` for each major
  - for tracking
- the driver also calls `cdev_init` and `cdev_add` to add a `cdev` to
  `cdev_map`
  - this decides the real fops
- for simpler case, the driver can call `register_chrdev` to do both
- for even simpler case, the driver can call `misc_register`

## Random Notes

- open fileoperation has prototype `int (*open) (struct inode *, struct file *)`
  `struct inode` is the file on the fs, while `struct file` is a fd
- char node major number is a limited resouce.  `register_chrdev_region` to
  reserve a range.
