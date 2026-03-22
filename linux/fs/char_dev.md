Character Devices
=================

## Overview

- a char dev is identified by its major/minor
  - `mknod` creates an inode with `def_chr_fops` as the fops
  - only major/minor matters; filename is irrelevant
- `def_chr_fops::open` points to `chrdev_open`
  - whenever the inode is opened, `chrdev_open` looks up the corresponding
    `cdev` from the global `cdev_map` using the major/minor
  - `filp->f_op` is replaced by `cdev->ops` and the driver takes over from
    here.
- global char major/minor
  - some subsystems/drivers are assigned static majors
    - `TTY_MAJOR` is 4
    - `LOOP_MAJOR` is 7
    - `DRM_MAJOR` is 226
    - `register_chrdev_region` claims a major and multiple minors
  - some subsystems/drivers allocate majors dynamically
    - `alloc_chrdev_region` allocs a major and multiple minors
      - `find_dynamic_major` returns an unused major from `234..254` or
        `384..511`
  - `char_device_struct` is purely internal
- global `cdev_map`
  - driver calls `cdev_add` to add a `cdev` to `cdev_map`
    - driver should have allocated/registered a major/minor
    - `cdev->ops` points to driver fops
    - this sets up a major/minor to `cdev` mapping
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

- driver should
  - call `register_chrdev_region` or `alloc_chrdev_region` to register/alloc a
    major and multiple minors
  - call `cdev_init` and `cdev_add` to add a `cdev` to global `cdev_map` for
    major/minor to `cdev` mapping
- `register_chrdev` is a convenient helper for those with static majors
  - `__register_chrdev_region` registers a major and multiple minors
  - `cdev_alloc` and `cdev_add` add a `cdev` to `cdev_map`
- `misc_register` is a convenient helper for those with a single minor
  - it will have `MISC_MAJOR` as the major and can alloc a minor dynamically

## Random Notes

- open fileoperation has prototype `int (*open) (struct inode *, struct file *)`
  `struct inode` is the file on the fs, while `struct file` is a fd
- char node major number is a limited resouce.  `register_chrdev_region` to
  reserve a range.
