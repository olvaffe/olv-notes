Kernel init
===========

## `start_kernel`

- `start_kernel` is the entry point of the comomn kernel code
  - only the boot cpu is up at this point
  - on x86-64, after some early assembly code, the boot cpu jumps to
    `x86_64_start_kernel` which calls `start_kernel`
  - on arm64, after some early assembly code, the boot cpu jumps to
    `start_kernel`
- The function performs various initializations.  The interesting ones are
  - `pr_notice("%s", linux_banner);` prints the banner
  - `setup_arch` does arch-specific initializations such as
    - calling `memblock_add` to add physical memories to memblock
    - calling `free_area_init` to initialize memory zones
  - `setup_command_line` saves away the cmdline
  - `setup_per_cpu_areas` initializes per-cpu areas
  - `trap_init` initializes cpu trap table
  - `mm_init` initializes memory allocators
    - `mem_init` is expected to call `memblock_free_all` to hand pages over to
      buddy allocator
    - `kmem_cache_init` initializes the slab allocator
    - `vmalloc_init` prepares for vmalloc/ioremap
  - `sched_init` initializes the scheduler
  - `workqueue_init_early` performs first stage of workqueue init
  - `rcu_init` initializes rcu
  - `early_irq_init` initializes the irq subsystem and calls
    `arch_early_irq_init`
  - `init_IRQ` initializes all hw irqs
  - `tick_init` initializes periodic kernel ticks
  - `init_timers` initializes timers
  - `hrtimers_init` initializes high-resolution timers
  - `softirq_init` initializes softirq subsystem
  - `timekeeping_init` initializes timekeeping (for `ktime_get*`)
  - `time_init` is arch-specific
    - `of_clk_init` to initialize `clock_provider`
    - `timer_probe` to scan devices and match drivers
      - drivers use `TIMER_OF_DECLARE` and `TIMER_ACPI_DECLARE`
  - `local_irq_enable` enables irq
  - `console_init` initializes console (for printk)
  - `acpi_early_init` initializes acpi
  - `late_time_init` is late `time_init`
  - `sched_clock_init` initializes jiffy-based clock
  - `pid_idr_init` initializes pid allocator
  - `anon_vma_init` initializes anon vma allocator
  - `fork_init` initializes task allocator
  - `vfs_caches_init` initializes various vfs caches as well as rootfs
    - `mnt_init` mounts `rootfs` to `/`
  - `pagecache_init` initializes pagecache
  - `signals_init` initializes signal subsystem
  - `proc_root_init` initializes procfs
  - `arch_call_rest_init` performs the rest initializations

## `arch_call_rest_init`

- `arch_call_rest_init` calls `rest_init` in the common kernel code
- `rest_init` forks off two tasks
  - pid 1 runs `kernel_init`
  - pid 2 runs `kthreadd`, which forks off kthreads in response to
    `kthread_create`
- it then calls `cpu_startup_entry` and enters `do_idle` loop
  - and the scheduler switches to pid 1 task

## `init_task`

- note that so far we run as pid 0 whose `task_struct` is statically-defined
  in `init/init_task.c`
  - `get_current` is arch-specific and archs set things up so that
    `get_current` returns `init_task` to this point
- `init_task`
  - `stack` is `init_stack`, which is in the data section (`INIT_TASK_DATA`)
  - `active_mm` is `init_mm`
  - `cred` and `real_cred` is `init_cred`
  - `comm` is `INIT_TASK_COMM`, which is `swapper`
  - `fs` is `init_fs`
  - `files` is `init_files`
  - `signal` is `init_signals`
  - `sighand` is `init_sighand`
  - `nsproxy` is `init_nsproxy`
  - `thread_pid` is `init_struct_pid` w/ pid 0

## `kernel_init`

- pid 1 runs `kernel_init`
  - it calls `kernel_init_freeable` to intialize more subsystems
  - it then `execve("/init")` or `execve("/sbin/init")` depending on whether
    there is initramfs
- `kernel_init_freeable` initializes more subsystems
  - these init functions are marked `__init` and will be freed
  - `smp_prepare_cpus` prepares non-boot cpus
  - `workqueue_init` initializes workqueue
  - `do_pre_smp_initcalls` runs `early_initcall` calls
  - `smp_init` brings up non-boot cpus
  - `page_alloc_init_late` does late-init for the buddy allocator
  - `do_basic_setup` does basic setup
    - `driver_init` initializes driver subsystems
    - `do_initcalls` runs all initcalls
  - `console_on_rootfs` opens `/dev/console` and makes it fd 0, 1, 2
  - if there is no initramfs, `prepare_namespace` mounts the root device to
    `/`, overriding `rootfs`

## rootfs

- during `start_kernel`, a `mnt_namespace` is created and a tmpfs is mounted
  - `vfs_caches_init` calls `mnt_init` which calls `init_mount_tree`
  - `vfs_kern_mount` creates a mount for `rootfs_fs_type`
    - `rootfs` is just a tmpfs and is empty
  - `alloc_mnt_ns` creates a `mnt_namespace`
  - `ns->root` is initialized to the rootfs mount
  - `init_task.nsproxy->mnt_ns` is initialized to `ns`
  - `set_fs_pwd` to chdir to the root inode of `rootfs`
  - `set_fs_root` to chroot to the root inode of `rootfs`
- pid 1 calls `populate_rootfs` or `default_rootfs` to populate rootfs
  - `kernel_init_freeable -> do_basic_setup -> do_initcalls`
  - if `CONFIG_BLK_DEV_INITRD` is defined, `populate_rootfs` unpacks initramfs
    to `rootfs`
    - when no initramfs is supplied at build time or by bootloader, there is a
      default initramfs created from `usr/default_cpio_list` at build time
  - otherwise, `default_rootfs` is called to create `/dev`, `/dev/console`,
    and `/root` in `rootfs`

## init

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

- `CONFIG_BLK_DEV_INITRD` enables initramfs/initrd support
- cpio archive
  - `usr/initramfs_data.S` defines `__initramfs_size` in `.init.ramfs` section
    - when `CONFIG_INITRAMFS_SOURCE` is specified, it includes the specified
      cpio archive
    - otherwise, it generates a cpio archive according to
      `usr/default_cpio_list`
    - the cpio archive is embedded in the kernel image
  - `INIT_RAM_FS` defines `__initramfs_start` and includes `.init.ramfs` section
    to help find the embedded archive
- cmdline
  - `initrd=` requests the bootloader to load an external cpio archive
    - `initrd_start` and `initrd_end` are set to the address of the loaded
      cpio archive in memory
  - `init=` is handled by `init_setup` and updates `execute_command`
    - default to `/sbin/init`
  - `rdinit=` is handled by `rdinit_setup` and updates `ramdisk_execute_command`
    - default to `/init`
  - `root=` is handled by `root_dev_setup` and updates `saved_root_name`
- `populate_rootfs` is called by `do_initcalls`
  - it unpacks the built-in initramfs at `__initramfs_start`
  - it then unpacks the external initramfs at `initrd_start`
    - when initrd= is given, the bootloader loads an external initramfs and
      updates `initrd_start` and `initrd_end`
    - an external initiramfs can be unpacked using
      `$ gunzip -c initramfs.img | cpio -idv`
  - if unpack of `initrd_start` failed and `CONFIG_BLK_DEV_RAM` is set, it
    assumes `initrd_start` points to a ramdisk image instead
- without `CONFIG_BLK_DEV_INITRD`, `default_rootfs` instead of
  `populate_rootfs` is called to populate a minimal rootfs
- pid 1
  - `kernel_init` kthread has pid 1 and runs `kernel_init`.  At the end, it
    `do_execve` init and become the userspace pid 1.  Before that...
  - `console_on_rootfs` opens `/dev/console` and dups to fd 0, 1, and 2
  - if initramfs contains `/init`,  `ramdisk_execute_command` is set to `/init`
    - otherwise, `prepare_namespace` sets `ROOT_DEV` according to `root=` and
      mounts the real rootfs
  - `run_init_process` calls `do_execve`
- initramfs `/init` is executed.  It parses the kernel cmdline and does many
  other things.  Among them,
  - it mounts the root
  - execs `/sbin/init` of the root device with the help of `switch_root`
  - add break=premount or break=postmount to spawn a shell

## packing initramfs

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
