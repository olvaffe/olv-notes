QEMU
====

## Install

- `pacman -S qemu-base qemu-ui-sdl qemu-ui-opengl qemu-hw-display-{virtio-gpu,virtio-gpu-gl,virtio-gpu-pci,virtio-gpu-pci-gl,virtio-vga,virtio-vga-gl}`
- `pacman -S qemu-system-aarch64`

## Get Source Code

- `git clone https://gitlab.com/qemu-project/qemu.git`
- `cd qemu`
- `git submodule init`
- `git submodule update --recursive`
  - `roms/QemuMacDrivers` is a guest VGA driver for Mac PPC
  - `roms/SLOF` is a OpenFirmware (standard developed by Sun) BIOS for PPC
  - `roms/edk2` is a UEFI/PI development environment
  - `roms/ipxe` is a PXE implementation
  - `roms/openbios` is a OpenFirmware implementation
  - `roms/opensbi` is a RISV-V SBI implementation
  - `roms/qboot` is a minimal x86 BIOS
  - `roms/qemu-palcode` is the firmware for Alpha
  - `roms/seabios` is an implementation of x86 BIOS and VGA BIOS
  - `roms/seabios-hppa` is a port for parisc/hppa
  - `roms/skiboot` is OpenPower Abstraction Layer (OPAL) for POWER
  - `roms/u-boot` is a bootloader for ARM, PPC, MIPS, etc.
  - `roms/u-boot-sam460ex` is a u-boot port for sam460ex
  - `roms/vbootrom` is a boorom for Nuvoton BMC
  - `tests/lcitool/libvirt-ci` is for CI
- meson subprojects
  - `subprojects/berkeley-softfloat-3` is a sw float impl for testing
  - `subprojects/berkeley-testfloat-3` is a test suite for float impl
  - `subprojects/dtc` is DeviceTree compiler
  - `subprojects/keycodemapdb` maps between key codes/symbols/etc
  - `subprojects/libvduse` is for sw-emulated vDPA device in userspace
  - `subprojects/libvhost-user` is for vhost in userspace
  - `subprojects/slirp` is a user-mode TCP/IP emulator

## Build

- `mkdir out; cd out`
- `../configure --target-list=x86_64-softmmu --enable-{debug,opengl,sdl,slirp,virglrenderer}`
  - it creates python venv under `pyvenv`
  - it ensures certain python modules are installed according to
    `pythondeps.toml`
  - it generates various `config*`
  - it invokes meson as `out/pyvenv/bin/meson setup \
      -Dopengl=enabled -Dsdl=enabled -Dslirp=enabled -Dvirglrenderer=enabled \
      --native-file config-meson.cross -Ddocs=disabled -Dplugins=true out`
  - or, `../configure --target-list=x86_64-softmmu --skip-meson` and run meson
    manually
- `ninja` builds qemu proper
- `make` builds qemu proper and firmwares
- meson options
  - `opengl` requires epoxy and, together with `sdl`, enables `-display sdl,gl=on`
  - `sdl` (or `gtk`) requires sdl and eanbles `-display sdl`
  - `slirp` enables userspace network emulation
  - `virglrenderer` requires virglrenderer and, together with `opengl`,
    enables `-device virtio-vga-gl`

## Example: GPT Disk Image

- `fallocate -l 4G test.img`
- `echo -e 'label:gpt\nsize=1G,type=uefi\ntype=linux' | sfdisk test.img`
- `losetup -P /dev/loop0 test.img`
- `mkfs.fat -F32 /dev/loop0p1`
- `mkfs.ext4 /dev/loop0p2`
- `mount /dev/loop0p2 /mnt`
  - arch
    - `mkdir -p /mnt/var/lib/pacman`
    - `pacman --root /mnt -Sy base`
    - `vi /mnt/etc/pacman.d/mirrorlist`
  - debian
    - `debootstrap --variant minbase testing /mnt`
- `umount /mnt`
- `losetup -D`
- `systemd-nspawn -i test.img`
  - arch
    - `pacman-key --init`
    - `pacman-key --populate`
    - `pacman -S linux vim`
    - `bootctl install`
    - `echo -e "linux /vmlinuz-linux\ninitrd /initramfs-linux-fallback.img\noptions console=ttyS0,115200 root=/dev/vda2 loglevel=7" > /boot/loader/entries/arch.conf`
      - `loglevel=7` because arch has `CONFIG_CONSOLE_LOGLEVEL_DEFAULT=4`
  - debian
    - `echo "console=ttyS0,115200 root=/dev/vda2 noresume" > /etc/kernel/cmdline`
      - `noresume` because debian initrd waits for swap for resume
    - `apt install linux-image-amd64 vim systemd-{resolved,timesyncd,boot}`
  - `echo 'root:test0000' | chpasswd`
  - `echo -e "/dev/vda2\t/\text4\trw\t0\t1\n/dev/vda1\t/boot\tvfat\trw\t0\t2" > /etc/fstab`
- `zstd test.img`

## Example: Power Up

- `qemu-system-x86_64 -machine q35 -accel kvm -cpu host -m 2G`
  - this powers on the machine with default devices
  - to escape mouse grab, `ctrl-alt-g`
- `qemu-system-x86_64 -machine q35 -accel kvm -cpu host -m 2G -nodefaults -nographic -serial mon:stdio`
  - this is without default devices and is headless
- `-nodefaults` disables default devices
  - this includes audio, serial, parallel, monitor, floppy, cdrom, vga, and net
  - serial, parallel, and monitor are typically dummy and unusable
- `-nographic` runs headless
  - it has several implications
    - `-machine q35,graphics=off` tells seabios to output to serial
    - `-vga none` disable vga emulation
    - `-display none` disables host window
    - `-serial mon:stdio` enables serial/parallel (if not disabled by `-nodefaults`)
  - to access monitor and exit qemu, `ctrl-a x`

## Example: Boot Kernel

- `-bios /usr/share/ovmf/OVMF.fd -drive file=test.img,format=raw,if=virtio`
  - `-bios` specifies OVMF instead of the default seabios at `pc-bios/bios.bin`
    - this is done because systemd-boot only supports uefi
    - a better way is to use `-pflash` to separate ovmf ro code and rw vars
  - remove `/boot/NvVars` on rootfs if uefi gets misconfigured
    - this happens whenever the drive is set up differently
- `-kernel ... -initrd ... -append ...` skips bootloader and boots kernel directly
  - this preloads specified kernel/initrd/cmdline into vm memory
  - this also enables `linuxboot.bin` option rom which boots the kernel directly
    - source code at `pc-bios/optionrom/linuxboot.S`

## Example: Slirp Networking

- `-nic user,model=virtio-net-pci`
  - this emulates a virtio-net pci card
  - it connects to a network where
    - gateway/firewall/dhcp is at 10.0.2.2
    - dns is at 10.0.2.3
- vm setup
  - `echo -e '[Match]\nName=en*\n[Network]\nDHCP=yes' > /etc/systemd/network/en.network`
  - `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`
  - `systemctl enable --now systemd-networkd systemd-resolved`

## Example: Tap Networking

- host setup
  - `sysctl net.ipv4.conf.all.forwarding=1`
  - nft
    - `nft flush ruleset`
    - `nft add table ip nat`
    - `nft add chain ip nat postrouting '{type nat hook postrouting priority srcnat;}'`
    - `nft add rule ip nat postrouting oifname <iface> masquerade`
  - tap
    - `ip tuntap add tap0 mode tap user $(whoami)`
    - `ip link set tap0 up`
    - `ip addr add 192.168.0.1/24 dev tap0`
  - dhcp
    - `dnsmasq -d --port 0 --listen-address 192.168.0.1 --bind-interfaces --dhcp-range 192.168.0.10,192.168.0.20`
      - or, use `systemd-networkd` with `DHCPServer=yes`
- `-nic tap,ifname=tap0,model=virtio-net-pci,script=no,downscript=no`
  - tx-checksumming seems broken with virtio-net-pci
    - remove `model=virtio-net-pci` or `ethtool -K tap0 tx-checksumming off`
      to work around
    - otherwise, dhcp offers from dnsmasq have bad checksums

## Example: Bridge Networking Between VMs

- host setup
  - bridge
    - `ip link add br0 type bridge`
    - `ip link set br0 up`
  - tap
    - `ip tuntap add tap0 mode tap user $(whoami)`
    - `ip tuntap add tap1 mode tap user $(whoami)`
    - `ip link set tap0 master br0`
    - `ip link set tap1 master br0`
    - `ip link set tap0 up`
    - `ip link set tap1 up`
- first vm with tap0
  - `ip address add 192.168.1.1/24 dev enp0s1`
    - or, set up forwarding/firewall/nat/dhcp/dns
- second vm with tap1
  - `ip address add 192.168.1.2/24 dev enp0s1`

## Example: AArch64

- `qemu-system-aarch64 -machine virt -accel tcg -cpu cortex-a76 -smp 2 -m 2G -nodefaults -nographic -serial mon:stdio`
  - `-machine virt,dumpdtb=qemu.dtb` to dump dtb
- `-kernel Image.gz -initrd initramfs.cpio.zst -append ...`
- `-display sdl,gl=core -device virtio-gpu-gl -nic user,model=virtio-net-pci`

## Bootstrap with ISO

- `fallocate -l 32GiB arch.img`
  - or, `./qemu-img create -f raw arch.img 32G`
  - qcow2 was used when filesystems did not support sparse files
- `MACHINE_OPTS="-machine q35 -cpu host -accel kvm -smp 2 -m 4G -nodefaults"`
- `DISPLAY_OPTS="-display sdl,gl=core -device virtio-vga-gl"`
- `BLOCK_OPTS="-drive file=arch.img,format=raw"`
- `NETWORK_OPTS="-nic user,hostfwd=tcp::2222-:22,model=virtio"`
- `./qemu-system-x86_64 \
   $MACHINE_OPTS $DISPLAY_OPTS $BLOCK_OPTS $NETWORK_OPTS \
   -drive file=archlinux-x86_64.iso,media=cdrom`
- common issues
  - `-smp 2`, otherwise arch iso refuses to boot to kernel
  - `ssh -p 2222 localhost` to ssh into the guest
    - the guest eth must be up (with dhcp?) first
- grub
  - if using bios with a gpt disk, grub expects the first partition to have
    type `BIOS boot` and size 1MB
    - followed by ESP and other partitions
  - `mkinitcpio -P`
  - `grub-install` and `grub-mkconfig`

## Tips

- release mouse grab
  - Ctrl-Alt or
  - Ctrl-Alt-G
- SSH
  - ssh ssh://root@localhost:2222
- headless
  - `-vnc :0`
  - `-nographic`
    - Ctrl-A X to quit qemu
- custom kernel
  - `-kernel` and `-append`

## Options

- there are always two parts of an emulated device
  - the emulated guest hardware, or the frontend
  - the backend in the host
- there are usually three ways to configure an emulated device
  - the legacy way: do not use
  - the new way: tedious
  - the convenient way
- `-nodefaults` to disable default devices
- Character devices
  - the new way
    - `-chardev TYPE,id=BLAH` for the backend
    - `-device TYPE,chardev=BLAH` for the frontend
    - example
      - `-chardev file,path=<path>,id=foo` to use a file as backend
      - `-device virtio-serial-pci` to add a frontend console bus
        - this is driven by `virtio_console` guest driver
      - `-device virtconsole,chardev=foo` to add a frontend console port
        - this port shows up as `/dev/vport0p0` in the guest
        - qemu also sends `VIRTIO_CONSOLE_CONSOLE_PORT`, which causes the port
          to be owned by hvc and is accessible as `/dev/hvc0`
      - or, `-device virtserialport,chardev=foo`
        - this port shows up and is accessible as `/dev/vport0p0` in the guest
    - another example
      - `-chardev stdio,mux=on,id=foo` to use stdio as backend
      - `-device isa-serial,chardev:foo` to connect guest `/dev/ttyS0` to the backend
      - `-mon chardev=foo` to connect monitor to the backend
  - the convenient way
    - `-serial mon:stdio`
- Network devices
  - the legacy way
    - `-net BACKEND` for the backend
    - `-net nic,model=MODEL` for the frontend
  - the new way
    - `-netdev BACKEND,id=foo` for the backend
    - `-device MODEL,netdev=foo` for the frontend
    - example
      - `-netdev user,id=foo` to use slirp as the backend
      - `-device virtio-net-pci,netdev=foo` to add frontend virtio-net pci
        - this is driven by `virtio_net` guest driver
  - the convenient way
    - `-nic BACKEND,model=MODEL`
    - example
      - `-nic user,model=virtio-net-pci`
- Block devices
  - the legacy way
    - `-hda <img>` expands to `-drive media=disk,index=0,file=<img>`
    - `-hdb <img>` expands to `-drive media=disk,index=1,file=<img>`
    - `-cdrom <img>` expands to `-drive media=cdrom,index=2,file=<img>`
    - `-fda <img>` expands to `-drive if=floppy,index=0,file=<img>`
    - `-mtdblock <img>` expands to `-drive if=mtd,file=<img>`
    - `-sd <img>` expands to `-drive if=sd,file=<img>`
    - `-pflash <img>` expands to `-drive if=pflash,file=<img>`
  - the new way
    - `-blockdev driver=TYPE,node-name=BLAH` for the backend
    - `-device TYPE,drive=BLAH` for the frontend
    - example
      - `-blockdev driver=file,node-name=foo,filename=<path>` for
        accessing the file
      - `-blockdev driver=raw,node-name=bar,file=foo` for using the file above
        as a raw image
      - `-device virtio-blk-pci,drive=bar` to add a frontend virtio-blk pci
        device
        - this is driven by `virtio_blk` guest driver
      - or, `-device ide-hd,drive=bar`, to add a frontend SATA disk on the
        SATA bus
  - the convenient way
    - `-drive file=<path>,format=raw,if=virtio`
- Input devices
  - the new way
    - `-display` for the backend
    - `-device virtio-keyboard-pci` to add a frontend keyboard
      - this is driven by `virtio_input` guest driver
    - `-device virtio-mouse-pci` to add a frontend mouse
- VGA
  - the legacy way
    - `-display` for the backend
    - `-vga` for the frontend
  - the new way
    - `-display` for the backend
    - `-device` for the frontend
    - example
      - `-display sdl,gl=on` for sdl/gl-based backend
      - `-device virtio-vga-gl` to add a frontend virtio-gpu card
        - this is driven by `virtio_gpu` guest driver
        - this is vga-compatible so it also works without the guest driver
      - or,
        - `-device virtio-vga`, with vga, no virgl
        - `-device virtio-gpu-pci`, no vga, no virgl
        - `-device virtio-gpu-gl-pci`, no vga, with virgl

## Console

- `-serial mon:stdio` uses stdio as guest serial console, with muxed monitor
- `ctrl-a h` for help
  - `ctrl-a x` to exit
  - `ctrl-a c` to toggle between guest console and muxed monitor
- upon monitor cmd, `monitor_read` reads the cmd and `handle_hmp_command`
  handles it
- `info` is handled by `hmp_info_help`
- `info qtree -b` is handled by `hmp_info_qtree`
  - `qbus_print` prints `sysbus_get_default` bus recursively
- `info qom-tree` is handled by `hmp_info_qom_tree`
  - `print_qom_composition` prints `qdev_get_machine` machine recursively

## QOM

- example: `kvmclock`
  - `type_init(kvmclock_register_types)` registers the type before `main`
    - `type_register_static` adds `kvmclock_info` to the type table
  - `kvmclock_create` creates a device during `pc_q35_init`
    - `qdev_new(name)` calls `object_new` to create a `TYPE_KVM_CLOCK`
      - `type_get_or_load_by_name` looks up the registered type
      - `object_new_with_type` creates the object
        - `type_initialize` inits the class (and base classes) on demand
          - `kvmclock_class_init`
          - `sysbus_device_class_init`
          - `device_class_init`
          - `object_class_init`
        - obj is allocated
        - `object_initialize_with_type` inits the obj
          - `device_initfn`
    - `qdev_realize_and_unref` realizes the qdev
      - `qdev_set_parent_bus` adds the qdev to the bus
      - `object_property_set_bool` sets `realized` to true
        - `device_set_realized` handles the change
          - `kvmclock_realize`
- `QObject` is a generic value
  - `QBool` has a `bool`
  - `QNum` has a discriminated i64/u64/double
  - `QString` has a c-string
  - `QList` has a list of `QObject`
  - `QDict` has a dict of `QObject`
- `object_class_property_add` adds a class prop
  - it allocs a `ObjectProperty` and adds it to `klass->properties`
  - `prop->set` sets the prop value
    - the prop value is wrapped in a `QObject`
    - the `QObject` is wrapped in a `qobject_input_visitor_new`
    - the set function extracts the underlying value from the visitor and
      updates the object
  - `prop->get` returns the prop value
    - the get function gets the underlying value from the object and wraps it
      in a `QObject`
    - the `QObject` is wrapped in a `qobject_output_visitor_new`
- `object_property_add` adds an object prop
  - it allocs a `ObjectProperty` and adds it to `obj->properties`
- `object_property_find` finds an `ObjectProperty`
  - `object_class_property_find` finds the class prop first, including in the
    base classes
  - it finds in `obj->properties` if there is no class prop
- convenience helpers
  - the prop set/get callbacks work with `Visitor`
  - `object_property_set_qobject` and `object_property_get_qobject` work with
    `QObject`
  - `object_property_set_str` and `object_property_get_str` work with string
    directly
  - same for other primitive types
- `object_property_add_alias` adds an object prop that aliases another object
  prop
- `object_property_add_child` adds an object prop and sets the value to
  another object
  - the other object is a child object and is stored directly in
    `prop->opaque`
  - `object_resolve_path_component` returns the child object directly
  - get returns `object_get_canonical_path` of the child object

## Startup

- most things in qemu are modules
- modules are registered before `main()` using `register_module_init`
  - each module type is added to a different list
- when qemu enters main, it calls `module_call_init` to initialize modules
- for a QOM (qemu object model) module, the initialization routine registers
  the object type with `type_register_static`
  - this adds the type to the typesystem
  - the type can be looked up using the type name
- for a OPTS module, the initialization routine registers `QemuOptsList` with
  `qemu_add_opts`
  - the core also registers many `QemuOptsList` itself
  - they are used to parse top-level options
- all top-level options are defined in `qemu-options.def`
  - to parse an option, qemu finds the corresponding registered `QemuOptsList`
    and calls `qemu_opts_parse`
  - it parses the option into a `QemuOpts` that can be looked up and used
    later
- qemu then picks a machine class
  - it gets a list of all registered machine types in QOM
  - it looks at the parsed machine option to pick the desired machine class
  - it initializes the QOM class
- qemu goes on to do many things, mainly to configure the machine according to
  the specified options
  - one of them is to add devices specified with device options to the machine
- when the machine configuration is finalized, qemu calls
  `machine_run_board_init` to initialize
  - this calls `pc_q35_init` for q35 machine
  - it adds a bunch more essential devices (each of a QOM type) to the machine

## Machine

- `DEFINE_Q35_MACHINE_AS_LATEST(10, 1)` defines `q35` machine
  - `pc_q35_machine_10_1_init` calls `pc_q35_init`
  - `pc_q35_machine_10_1_class_init`
    - `pc_q35_machine_10_1_options`
      - `m->default_machine_opts = "firmware=bios-256k.bin"`
      - `m->default_display = "std"`
      - more
    - `mc->init = pc_q35_machine_10_1_init`
    - `mc->is_default = false`
    - `mc->alias = "q35"`
  - `pc_q35_machine_10_1_info`
    - `.name = "pc-q35-10.1-machine"`
    - `.parent = TYPE_PC_MACHINE`
    - `.class_init = pc_q35_machine_10_1_class_init`
  - `pc_q35_machine_10_1_register`
    - `type_register_static(&pc_q35_machine_10_1_info)`
  - `type_init(pc_q35_machine_10_1_register)` expands to a ctor that calls
    `register_module_init`
- `qemu_init`
  - `qemu_init_subsystems` inits various modules
    - this includes `pc_q35_machine_10_1_register` which calls
      `type_register_static` to register `pc_q35_machine_10_1_info`
  - `qemu_create_machine` creates the machine
    - `qdict` is parsed from `-machine ...`
    - with `-machine q35`, `select_machine` selects q35
      - `object_class_get_list` calls `type_initialize` on all `TYPE_MACHINE`
        - `pc_q35_machine_10_1_class_init` inits the machine class
      - `find_machine` returns the machine class
    - `object_new_with_class` creates a `PCMachineState`
  - if `-nodefaults`, `qemu_disable_default_devices` disables default devices
  - `qemu_setup_display` sets up display
  - `qemu_create_default_devices` creates default devices
  - `qemu_apply_machine_options` calls `object_set_properties_from_keyval` to
    add `-machine ...` as properties
  - `qmp_x_exit_preconfig`
    - `qemu_init_board`
      - `machine_run_board_init` calls `pc_q35_machine_10_1_init`
        - `pcms->pcibus` is pci host bus, with this inheritance
          - `TYPE_Q35_HOST_DEVICE`
          - `TYPE_PCIE_HOST_BRIDGE`
          - `TYPE_PCI_HOST_BRIDGE`
          - `TYPE_SYS_BUS_DEVICE`
          - `TYPE_DEVICE`
          - `TYPE_OBJECT`
        - `lpc` is a pci device that provides isa bus
          - it is a `TYPE_ICH9_LPC_DEVICE`
          - it has a `TYPE_MC146818_RTC`
        - `x86ms->rtc` is rtc on isa bus
        - `pc_i8259_create` creates `TYPE_KVM_I8259` on isa bus
        - `pc_basic_device_init`
          - hpet is `TYPE_HPET`
          - `kvm_pit_init` creates `TYPE_KVM_I8254` timer
          - pcspk
          - `pc_superio_init` creates `TYPE_I8042`, vmmouse, etc.
        - ich9 sata controller `TYPE_ICH9_AHCI`
        - smbus `TYPE_ICH9_SMB_DEVICE`
        - `pc_vga_init` creates vga
          - if `-vga virtio`, `TYPE_VIRTIO_VGA`
          - if `-nodefaults`, none
        - `pc_nic_init` creates nic
          - if `-nodefaults`, none
    - `qemu_create_cli_devices` calls `device_init_func` for each `-device`
      - `qdev_device_add_from_qdict` adds the device
      - for `-device virtio-net`,
        - `qdev_get_device_class` uses `qdev_alias_table` to map `virtio-net`
          to `virtio-net-pci`
        - `virtio_net_pci_instance_init` inits the dev
        - `virtio_net_pci_realize` realizes
      - for `-device virtio-vga-gl`,
        - `virtio_vga_gl_inst_initfn` inits the dev
        - `virtio_gpu_gl_device_realize` realizes

## Display

- Before main, type `virtio-vga` is registered to the QOM
- `-vga virtio` selects the `virtio-vga` as the VGA type for the machine
- `-display` does not use QOM or `QemuOpts`
  - displays are registered with `qemu_display_register`
  - the option is parsed to `DisplayOptions`
- once a display type is selected, the display and the console are early
  initalized with `qemu_display_early_init` and `qemu_console_early_init`
- when the q35 machine is initialized in `machine_run_board_init`, it calls
  `pc_vga_init`
  - this creates the `virtio-vga` device
  - it calls `virtio_vga_base_realize`, and indirect calls
    `virtio_gpu_device_realize` to realize it vgpu object
  - virtio-gpu registers a graphic console with `graphic_console_init`
- finally `qemu_display_init` is called
  - this creates the window for the VM

## virtio-input

- `-device virtio-keyboard-pci` and ` -device virtio-mouse-pci`
  - when devices are added to the machine, the options add the desired PCI devices
  - each of them own a `virtio-{keyboard,mouse}-device`
  - they register input handlers with `qemu_input_handler_register`
  - when the display gets inputs, it invokes the handlers.  This gives them a
    chance to inject the events to the device and sends IRQs
- when a key is pressed, it is processed by the display (SDL2/gtk/...) and
  `qkbd_state_key_event` is called
  - it goes a long way but eventually `qemu_input_event_send_impl` is called

## main loop

- `main_loop` is pretty standard
  - if there is any timer, timeout is set to the diff to the next timer;
    otherwise, timeout is indefinite
  - it unlocks the global iothread lock with `qemu_mutex_unlock_iothread`
  - it polls fds with timer timeout
  - after waking up, it grabs the global lock with `qemu_mutex_lock_iothread`
  - it processes all polled fds with pending works
  - it runs all of its timers with `qemu_clock_run_all_timers`
    - `text_console_update_cursor` wakes up every `CONSOLE_CURSOR_PERIOD / 2`
      (250) ms
    - `gui_update` wakes up every `GUI_REFRESH_INTERVAL_DEFAULT` or
      `GUI_REFRESH_INTERVAL_IDLE` ms (to process input)
