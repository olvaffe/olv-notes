Kernel UML
==========

## Overview

- <https://docs.kernel.org/virt/uml/user_mode_linux_howto_v2.html>
- `main` in `os-Linux/main.c` is the entrypoint
- enable gdb support
  - `CONFIG_DEBUG_KERNEL=y`
  - `CONFIG_DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT=y`

## Tasks

- when `kernel_clone` is called to fork, `copy_thread` is called
  - it updates `p->thread.switch_buf` to point to `new_thread_handler` when
    jumped to
  - `new_thread_handler` will call the task entrypoint
    - `kernel_init` for pid 1
    - `kthreadd` for pid 2
- when `__schedule` is called to reschedule,
  - `context_switch` calls `__switch_to`
  - `switch_threads` jumps to `p->thread.switch_buf`
- tasks and context switches are thus simulated by `setjmp` and `longjmp`

## Boot

- early boot
  - `main` calls `linux_main`
  - `linux_main` calls `start_uml`
    - `os_early_checks` performs early checks
    - the memory layout is initialized
      - `uml_physmem` is the start addr of the kernel image
      - `uml_reserved` is the end addr of reserved memory
      - `physmem_size` is the physical mem size, defaults to 64MB
  - `start_uml` calls `start_kernel_proc`
    - it updates `init_task.thread.switch_buf` (pid 0) to `uml_finishsetup`
      and jumps to it
    - `uml_finishsetup` performs post setup and calls `new_thread_handler` to
      execute as pid 0, whose entrypoint is `start_kernel_proc`
  - `start_kernel_proc` calls `start_kernel`
- `setup_arch`
  - `setup_physmem` allocates the memory for physical mem
    - `physmem_fd` is `MAP_FIXED` to `reserve_end`
    - the memory is added to memblock
  - `paging_init` sets up paging
- `mem_init` frees all phy memory from memblock to buddy allocator
- `kernel_init` calls `kernel_init_freeable` which calls `do_basic_setup`
  which calls initcalls
  - `um_pci_init` bails unless `CONFIG_UML_PCI_OVER_VIRTIO_DEVICE_ID`
    - but it still calls `logic_iomem_add_region` which `request_resource`
