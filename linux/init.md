Kernel init
===========

## PID 0: `start_kernel`

- `init_task` has PID 0 and is statically-defined in `init/init_task.c`
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
- arch ensures `get_current` returns `init_task` initially
  - x86 `get_current` calls `this_cpu_read_stable(current_task)`, where
    `current_task` is initialized in
    `DEFINE_PER_CPU_CACHE_HOT(struct task_struct *, current_task) = &init_task;`
  - arm64 `get_current` reads `sp_el0` reg, where `__primary_switched` calls
    `init_cpu_task` to init to `init_task`
- `start_kernel` is the entry point of the comomn kernel code
  - only the boot cpu is up at this point
  - on x86-64, after some early assembly code, the boot cpu jumps to
    `x86_64_start_kernel` which calls `start_kernel`
  - on arm64, after some early assembly code, the boot cpu jumps to
    `start_kernel`
- The function performs various initializations.  The interesting ones are
  - `local_irq_disable()` disables irq during early boot
  - `boot_cpu_init` marks the boot cpu ready
  - `pr_notice("%s", linux_banner);` prints the banner
  - `setup_arch` performs arch-specific inits and returns cmdline
    - `parse_early_param` parses `early_param` such as `earlycon`
    - `memblock_add` adds physical memories to memblock
    - `free_area_init` inits memory zones
    - many more
  - `setup_command_line` saves cmdline to `saved_command_line`
  - `setup_per_cpu_areas` inits per-cpu areas
  - `boot_cpu_hotplug_init` marks the boot cpu online
  - `pr_notice("Kernel command line: %s\n", saved_command_line)` prints the
    cmdline
  - `parse_early_param` parses `early_param` again
    - it is nop because `setup_arch` already calls it
  - `parse_args` parses all cmdline args
  - `setup_log_buf` allocs printk buf
  - `trap_init` inits cpu trap table
  - `mm_core_init` inits memory allocators
    - `memblock_free_all` hands pages over to buddy allocator
    - `kmem_cache_init` inits the slab allocator
    - `vmalloc_init` prepares for vmalloc/ioremap
  - `ftrace_init` inits ftrace
  - `sched_init` inits the scheduler
  - `workqueue_init_early` performs first stage of workqueue init
  - `rcu_init` inits rcu
  - `trace_init` inits tracing
  - `early_irq_init` inits the irq subsystem and calls
    `arch_early_irq_init`
  - `init_IRQ` inits all hw irqs
  - `tick_init` inits periodic kernel ticks
  - `init_timers` inits timers
  - `hrtimers_init` inits high-resolution timers
  - `softirq_init` inits softirq subsystem
  - `timekeeping_init` inits timekeeping (for `ktime_get*`)
  - `time_init` is arch-specific
    - `of_clk_init` to init `clock_provider`
    - `timer_probe` to scan devices and match drivers
      - drivers use `TIMER_OF_DECLARE` and `TIMER_ACPI_DECLARE`
  - `local_irq_enable` enables irq
  - `console_init` inits console (for printk)
    - this calls `console_initcall` initcalls
    - vt's `con_init` prints `Console: colour dummy device 80x25` and calls
      `register_console`
      - unless `keep_bootcon`, this also disables earlycon
  - `acpi_early_init` inits acpi
  - `late_time_init` is deferred `time_init`
    - on x86, it points to `x86_late_time_init`
  - `sched_clock_init` inits jiffy-based clock
  - `calibrate_delay` computes BogoMIPS
  - `arch_cpu_finalize_init` late-inits the boot cpu
  - `pid_idr_init` inits pid allocator
  - `anon_vma_init` inits anon vma allocator
  - `efi_enter_virtual_mode` tells efi to enter virtual mode
  - `fork_init` inits task allocator
  - `security_init` inits lsm
  - `net_ns_init` inits net
  - `vfs_caches_init` inits various vfs caches as well as rootfs
    - `mnt_init` mounts `rootfs` to `/`
  - `pagecache_init` inits pagecache
  - `signals_init` inits signal subsystem
  - `proc_root_init` inits procfs
  - `cpuset_init` inits cpuset
  - `cgroup_init` inits cgroup
  - `acpi_subsystem_init` inits acpi again
  - `rest_init` forks off two tasks and enters idle loop
    - pid 1 runs `kernel_init`, which ultimately execs `/sbin/init`
    - pid 2 runs `kthreadd`, which forks off kthreads in response to
      `kthread_create`
    - `cpu_startup_entry(CPUHP_ONLINE)` enters `do_idle` loop
- `do_idle` loop
  - the boot cpu runs `init_task` and is in an infinite loop calling `do_idle`
  - if there is no other task, the boot cpu enters idle states
  - if there is another task (e.g., pid 1 during boot), `schedule_idle`
    switches to the task
    - this changes the stack, and when the boot cpu returns from
      `schedule_idle`, it jumps to the prior frame of the new stack (e.g.,
      `kernel_init` of pid 1 during boot)
  - when the other task switches back to `init_task` and returns, the boot cpu
    jumps back to `do_idle` right after `schedule_idle`

## PID 1: `kernel_init`

- pid 1 runs `kernel_init`
  - it calls `kernel_init_freeable` to intialize more subsystems
  - it then `execve("/init")` or `execve("/sbin/init")` depending on whether
    there is initramfs
- `kernel_init_freeable` inits more subsystems
  - these init functions are marked `__init` and will be freed
  - `smp_prepare_cpus` prepares non-boot cpus
    - x86 calls `native_smp_prepare_cpus`
  - `workqueue_init` inits workqueue
  - `do_pre_smp_initcalls` runs `early_initcall` calls
  - `smp_init` brings up non-boot cpus
    - `idle_threads_init` forks off per-cpu idle tasks as threads of PID 0
    - `cpuhp_threads_init` forks off per-cpu hotplug tasks
    - `bringup_nonboot_cpus` brings up non-boot cpus
      - on x86, `secondary_startup_64` is the entry point
        - at the end, it calls `initial_code` which points to
          `start_secondary`
  - `sched_init_smp` inits scheduler for SMP
  - `page_alloc_init_late` does late-init for the buddy allocator
  - `do_basic_setup` does basic setup
    - `driver_init` inits driver subsystems
    - `init_irq_proc` creates `/proc/irq`
    - `do_initcalls` runs all initcalls
      - this includes `rootfs_initcall` which populates rootfs
  - `console_on_rootfs` opens `/dev/console` and makes it fd 0, 1, 2
  - if there is no `/init` (no initramfs), `prepare_namespace` mounts the root
    device to `/`, overriding rootfs
- pid 1 execs `/init` or `/sbin/init` at the end of `kernel_init`
  - it attemps to exec `ramdisk_execute_command` (`/init`)
    - with initramfs, the cpio archive provides `/init`
  - otherwise, it attemps `/sbin/init`
    - without initramfs, `prepare_namespace` has mounted the real root
  - `run_init_process` calls `kernel_execve`
    - `argv_init = { "/init", NULL }`
    - `envp_init = { "HOME=/", "TERM=linux", NULL }`
  - initramfs `/init` typically parses the kernel cmdline and does many
    things.  Among them,
    - it mounts the real root
    - execs `/sbin/init` of the real root with the help of `switch_root`
    - the script may understand `break=premount` or `break=postmount` to spawn
      a debug shell

## command line

- arch saves the cmdline to `boot_command_line`
  - on x86, `x86_64_start_kernel` calls `copy_bootdata`
    - it initializes `boot_params` and copies the cmdline to
      `boot_command_line`
    - later in `setup_arch`, if there is a built-in cmdline
      (`CONFIG_CMDLINE`), the built-in cmdline either replaces the boot
      cmdline or gets prepended
  - on arm64, `setup_arch` calls `setup_machine_fdt` which calls
    `early_init_dt_scan`
    - `early_init_dt_scan_chosen` copies `bootargs` from dt to
      `boot_command_line`
    - if there is a built-in cmdline (`CONFIG_CMDLINE`), the built-in cmdline
      either
      - replaces the boot cmdline if `CONFIG_CMDLINE_FORCE`,
      - is ignored if there is a boot cmdline, or
      - is used if there is no boot cmdline
- arch calls `parse_early_param` to parse `early_param`
  - `__setup_param` defines an entry in `.init.setup` section
  - `INIT_SETUP` puts them between `__setup_start` and `__setup_end`
- `start_kernel`
  - `setup_boot_config` parses `bootconfig`
  - `setup_command_line` aggregates cmdlines from different sources
  - `parse_args` parses the cmdline again
    - `__start___param` and `__stop___param`
      - `module_param` defines an entry in `.modinfo` section
      - `RO_DATA` puts them between `__start___param` and `__stop___param`
    - if a param is unknown at this point (non-early and non module-param;
      iow, defined by `__setup`, it is handled by `unknown_bootoption`
      - e.g., `console=` is handled by `unknown_bootoption`
- minimal cmdline
  - `root=` is required, whether kernel or initramfs mounts it
    - `root_dev_setup` parses `root=` to `saved_root_name`
    - `prepare_namespace` mounts `saved_root_name` when there is no initramfs
      - `parse_root_device` parses `saved_root_name` to `ROOT_DEV`
      - `mount_root` mounts the root
  - `rootwait` is required if the root device is probed asynchronously
    - it causes `wait_for_root` to be called to wait indefinitely until the
      device shows up
  - `ro`/`rw` is not needed
    - `ro` is the default
    - `systemd-remount-fs` will remount `/` according to `/etc/fstab`
  - `quiet`/`debug` is optional
    - `console_loglevel` defaults to 7 (`CONFIG_CONSOLE_LOGLEVEL_DEFAULT`)
    - `debug` sets it to 10 (`CONSOLE_LOGLEVEL_DEBUG`)
    - `quiet` sets it to 4 (`CONFIG_CONSOLE_LOGLEVEL_QUIET`)
  - `console=ttyS0,115200 console=tty0` is optional
    - printk logs to all `console=`
    - `/dev/console` is the last `console=`
    - when none specified, printk picks the first capabie device as the console

## initcall

- macros
  - `___define_initcall` defines a `initcall_t` in section `<sec>.init`
  - `console_initcall` uses section `.con_initcall.init`
  - `early_initcall` uses section `.initcallearly.init`
  - `pure_initcall` uses section `.initcall0.init`
  - `core_initcall` uses section `.initcall1.init`
  - `core_initcall_sync` uses section `.initcall1s.init`
  - `postcore_initcall` uses section `.initcall2.init`
  - `postcore_initcall_sync` uses section `.initcall2s.init`
  - `arch_initcall` uses section `.initcall3.init`
  - `arch_initcall_sync` uses section `.initcall3s.init`
  - `subsys_initcall` uses section `.initcall4.init`
  - `subsys_initcall_sync` uses section `.initcall4s.init`
  - `fs_initcall` uses section `.initcall5.init`
  - `fs_initcall_sync` uses section `.initcall5s.init`
  - `rootfs_initcall` uses section `.initcallrootfs.init`
  - `device_initcall` uses section `.initcall6.init`
    - `__initcall` is a shorthand for `device_initcall`
  - `device_initcall_sync` uses section `.initcall6s.init`
  - `late_initcall` uses section `.initcall7.init`
  - `late_initcall_sync` uses section `.initcall7s.init`
- `vmlinux.lds.h`
  - `CON_INITCALL` defines `__con_initcall_{start,end}` for
    `.con_initcall.init`
  - `INIT_CALLS` defines `__initcall_{start,end}` for `.initcall*.init`
    - also these symbols for each level in order
      - `__initcall0_start`
      - `__initcall1_start`
      - `__initcall2_start`
      - `__initcall3_start`
      - `__initcall4_start`
      - `__initcall5_start`
      - `__initcallrootfs_start`
      - `__initcall6_start`
      - `__initcall7_start`
    - within each level, sync initcalls are arranged after normal initcalls
      - the idea is that a user can use the normal initcall to queue a work
        and use the sync variant to wait for it
- `console_init` calls all `console_initcall`
  - it is called early from `start_kernel` by pid 0
  - they call `register_console` to register consoles
- `do_pre_smp_initcalls` calls all `early_initcall`
  - it is called from `kernel_init_freeable` by pid 1 before `smp_init`
- `do_initcalls` calls the rest
  - it is called from `kernel_init_freeable` by pid 1 almost last

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
  - `CONFIG_BLK_DEV_INITRD` enables initramfs/initrd support
  - if `CONFIG_BLK_DEV_INITRD` is defined, `populate_rootfs` unpacks initramfs
    to `rootfs`
    - it unpacks the built-in initramfs at `__initramfs_start`
      - when no custom initramfs is supplied at build time by
        `CONFIG_INITRAMFS_SOURCE`, there is a default initramfs created from
        `usr/default_cpio_list` at build time
    - it then unpacks the external initramfs at `initrd_start`
      - when initrd= is given, the bootloader loads an external initramfs and
        updates `initrd_start` and `initrd_end`
    - if unpack of `initrd_start` failed and `CONFIG_BLK_DEV_RAM` is set, it
      assumes `initrd_start` points to a ramdisk image instead
  - otherwise, `default_rootfs` is called to create `/dev`, `/dev/console`,
    and `/root` in `rootfs`
- `rootfs` is never unmounted
  - without initramfs, `prepare_namespace` mounts the real root to override
    rootfs
    - `parse_root_device` parses `root=` to get root device major:minor and
      save it to `ROOT_DEV`
    - `mount_root` mounts `ROOT_DEV` to `/root`
      - it creates `/dev/root` pointing to `ROOT_DEV`
      - `mount_root_generic` mounts `/dev/root` to `/root` and `chdir` to it
        - on success, dmesg prints `VFS: Mounted root (%s filesystem)%s on device %u:%u.\n`
        - on failure, dmesg prints some info and `VFS: Unable to mount root fs on %s`
    - `devtmpfs_mount` mounts devtmpfs to `dev` (relative to `/root`)
    - `init_mount` moves the mount from `.` to `/`
    - `init_chroot` chroots to `.`
  - with initramfs, `Documentation/filesystems/ramfs-rootfs-initramfs.rst`
    says

    When switching another root device, initrd would pivot_root and then
    umount the ramdisk.  But initramfs is rootfs: you can neither pivot_root
    rootfs, nor unmount it.  Instead delete everything out of rootfs to free
    up the space (find -xdev / -exec rm '{}' ';'), overmount rootfs with the
    new root (cd /newmount; mount --move . /; chroot .), attach
    stdin/stdout/stderr to the new /dev/console, and exec the new init
  - <https://git.kernel.org/pub/scm/libs/klibc/klibc.git/tree/usr/kinit/run-init>
  - <https://git.kernel.org/pub/scm/utils/util-linux/util-linux.git/tree/sys-utils/switch_root.c>
  - <https://git.busybox.net/busybox/tree/util-linux/switch_root.c>
  - we can't see `rootfs` in `/proc/mounts` because it is outside of chroot
    (the real root) and `show_vfsmnt` returns `SEQ_SKIP` for it
- built-in cpio archive
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

## packing/unpacking initramfs

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
- to create the cpio archive
  - `find . | cpio -o -H newc -R root:root > ../initramfs.cpio`
  - the kernel initramfs unpacker checks if the cpio starts with "070701",
    that is, the new portable format
- to unpack the cpio archive
  - `cpio -idmv < ../initramfs.cpio`
- unpack system initramfs
  - system initramfs often consist of an uncompressed cpio archive and a
    compressed cpio archive
    - the first archive holds the cpu microcode
  - `(cpio -idmv; zcat | cpio -idmv) < /boot/initramfs.img`
