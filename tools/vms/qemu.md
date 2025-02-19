QEMU
====

## Get Source Code

- `git clone https://git.qemu.org/git/qemu.git`
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
- `../configure --target-list=x86_64-softmmu --enable-{opengl,sdl,slirp,virglrenderer}`
  - it creates python venv under `pyvenv`
  - it ensures certain python modules are installed according to
    `pythondeps.toml`
  - it generates various `config*`
  - it invokes meson as `out/pyvenv/bin/meson setup \
      -Dopengl=enabled -Dsdl=enabled -Dslirp=enabled -Dvirglrenderer=enabled \
      --native-file config-meson.cross -Ddocs=disabled -Dplugins=true out`
- `ninja` builds qemu proper
- `make` builds qemu proper and firmwares
- meson options
  - `opengl` requires epoxy and enables `-display sdl,gl=on`
  - `sdl` (or `gtk`) requires sdl and eanbles `-display sdl`
  - `slirp` enables userspace network emulation
  - `virglrenderer` requires virglrenderer and enables `-vga virtio`

## Examples

- create gpt disk image
  - `fallocate -l 4G test.img`
  - `echo -e 'label:gpt\nsize=260M,type=uefi\ntype=linux' | sfdisk test.img`
  - `losetup -P /dev/loop0 test.img`
  - `mkfs.fat -F32 /dev/loop0p1`
  - `mkfs.ext4 /dev/loop0p2`
  - `mount /dev/loop0p2 /mnt`
  - arch
    - `mkdir -p /mnt/var/lib/pacman`
    - `pacman --root /mnt -Sy base`
    - `vi /mnt/etc/pacman.d/mirrorlist`
  - `umount /mnt`
  - `losetup -D`
  - `systemd-nspawn -i test.img`
    - `pacman-key --init`
    - `pacman-key --populate`
    - `pacman -S linux vim`
    - `echo 'root:test0000' | chpasswd`
    - `bootctl install`
    - `echo -e "linux /vmlinuz-linux\ninitrd /initramfs-linux-fallback.img\noptions console=ttyS0,115200 loglevel=7 root=/dev/sda2" > /boot/loader/entries/arch.conf`
    - `echo -e "/dev/sda2\t/\text4\trw\t0\t1" > /etc/fstab`
    - `echo -e "/dev/sda1\t/boot\tvfat\trw\t0\t2" >> /etc/fstab`
- first boot
  - `qemu-system-x86_64 -accel kvm -cpu host -m 2G -bios /usr/share/edk2/x64/OVMF.4m.fd -serial mon:stdio -drive file=test.img,format=raw -nodefaults -nographic`
- slirp networking: `-nic user,model=virtio-net-pci`
  - qemu emulates
    - gateway/firewall/dhcp: `10.0.2.2`
    - dns: `10.0.2.3`
  - `echo -e '[Match]\nName=*\n[Network]\nDHCP=yes' > /etc/systemd/network/all.network`
  - `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`
  - `systemctl enable --now systemd-networkd systemd-resolved`

## Bootstrap with ISO

- `fallocate -l 32GiB arch.img`
  - or, `./qemu-img create -f raw arch.img 32G`
  - qcow2 was used when filesystems did not support sparse files
- `MACHINE_OPTS="-machine q35 -cpu host -accel kvm -smp 2 -m 4G -nodefaults"`
- `DISPLAY_OPTS="-display sdl,gl=core -device virtio-vga-gl"`
- `BLOCK_OPTS="-hda arch.img"`
- `NETWORK_OPTS="-nic user,hostfwd=tcp::2222-:22,model=virtio"`
- `./qemu-system-x86_64 \
   $MACHINE_OPTS $DISPLAY_OPTS $BLOCK_OPTS $NETWORK_OPTS \
   -cdrom <cd>.iso -boot d`
- common issues
  - `-smp 2`, otherwise arch iso refuses to boot to kernel
  - `ssh -p 2222 localhost` to ssh into the guest
    - the guest eth must be up (with dhcp?) first

## Bootstrap with chroot

- partition and format
  - `fdisk arch.img`
    - if using bios with a gpt disk, grub expects the first partition to have
      type `BIOS boot` and size 1MB
    - followed by ESP and other partitions
  - `losetup -fP arch.img`
  - `mkfs.ext4 /dev/loop0p3`
  - `mount /dev/loop0p3 /mnt`
  - populate `/mnt`
  - `mkfs.vfat -F32 /dev/loop0p2`
  - `mount /dev/loop0p2 /mnt/boot`
- chroot
  - edit `/mnt/etc/pacman.d/mirrorlist` and maybe `/mnt/etc/resolv.conf`
  - `unshare -m chroot /mnt`
  - `mount -t proc none proc`
  - `mount -t devtmpfs none /dev`
  - `mount -t sysfs none /sys`
- install packages
  - `pacman-key --init`
  - `pacman-key --populate`
  - `pacman -Syu`
  - `pacman -S grub linux vim openssh`
- prepare network
  - `cat > /etc/systemd/network/ether.network <<EOF`

    [Match]
    Type=ether
    [Network]
    DHCP=yes
    EOF
  - `systemctl enable sshd`
- final touch
  - `passwd`
  - `useradd -m -G wheel olv`
  - `passwd olv`
  - `visudo`
  - `pkill gpg-agent`
- install grub (outside of chroot; do not work)
  - `grub-install --boot-directory=/mnt/boot --target=i386-pc /dev/loop0`
    - this seems ok, but requires installing bios-version of grub in the
      host
  - `grub-mkconfig -o /mnt/boot/grub/grub.cfg`
    - this script does not work that way
    - it should be run inside chroot and require `device.map`
- install grub (using qemu)
  - copy kernel and initramfs out of chroot
  - umount chroot and take down loop0
    - if chroot or loop0 is busy, see if pacman's `gpg-agent` is still alive
  - start qemu with
    - `-nographic -kernel vmlinuz-linux -initrd initramfs-linux-fallback.img
       -append 'console=ttyS0 loglevel=8 root=/dev/sda3 rw'`
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
    - `-device TYPE,chardev=BLAH` for the frontend
    - `-chardev TYPE,id=BLAH` for the backend
  - the convenient way
    - `-serial`
  - example
    - `-device virtio-serial` to add a frontend bus
    - `-device virtserialport,chardev=BLAH` to add the frontend
    - `-chardev socket,path=FOO,id=BLAH`
- Network devices
  - the legacy way
    - `-net nic,model=MODEL` for the frontend
    - `-net BACKEND` for the backend
  - the new way
    - `-device MODEL,netdev=BLAH` for the frontend
    - `-netdev BACKEND,id=BLAH` for the backend
  - the convenient way
    - `-nic BACKEND,model=MODEL`
  - example
    - `-nic user,model=virtio-net-pci`
- Block devices
  - the old way
    - `-drive if=TYPE,format=qcow2,file=FOO`
  - the new way
    - `-device TYPE,drive=BLAH` for the frontend
    - `-blockdev driver=TYPE,node-name=BLAH` for the backend
  - the convenient way
    - `-hda`, `-cdrom`, `-pflash`
  - example
    - `-device ide-hd,drive=BLAH` to add the frontend disk
    - `-blockdev driver=file,node-name=FILE,filename=PATH_TO_IMAGE` for
      accessing the file
    - `-blockdev driver=qcow2,node-name=BLAH,file=FILE` for accessing the file
      as a qcow2 image
  - example
    - `-device virtio-scsi-pci` to add a frontend SCSI HBA
    - `-device scsi-hd,drive=BLAH` to add the frontend disk
    - `-blockdev driver=file,node-name=BLAH,filename=PATH_TO_FILE` for the
      backend
- Input devices
  - `-device virtio-keyboard-pci`
  - `-device virtio-mouse-pci`
- VGA
  - 

## Starup

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
  -
