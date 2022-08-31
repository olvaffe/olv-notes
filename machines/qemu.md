QEMU
====

## Get Source Code

- `git clone https://git.qemu.org/git/qemu.git`
- `cd qemu`
- `git submodule init`
- `git submodule update --recursive`
  - `capstone` is a disassembly framework
    `https://github.com/aquynh/capstone`
  - `dtc` is DeviceTree compiler
    `https://git.kernel.org/pub/scm/utils/dtc/dtc.git`
  - `slirp` is a user-mode TCP/IP emulator
    `https://gitlab.freedesktop.org/slirp/libslirp`
  - `ui/keycodemapd` maps between key codes/symbols/etc and is owned by QEMU
  - `roms/QemuMacDrivers` is a guest VGA driver for Mac PPC
     `https://github.com/ozbenh/QemuMacDrivers`
  - `roms/SLOF` is a OpenFirmware (standard developed by Sun) BIOS for PPC
    `https://github.com/aik/SLOF/`
  - `roms/edk2` is a UEFI/PI development environment
    `https://github.com/tianocore/edk2`
  - `roms/ipxe` is a PXE implementation
    `git://git.ipxe.org/ipxe.git`
  - `roms/openbios` is a OpenFirmware implementation
    `https://github.com/openbios/openbios`
  - `roms/openhackware` is a (deprecated?) PowerPC Reference Platform (PReP)
    implementation
  - `roms/opensbi` is a RISV-V SBI implementation
    `https://github.com/riscv/opensbi`
  - `roms/qboot` is a minimal x86 BIOS
    `https://github.com/bonzini/qboot`
  - `roms/qemu-palcode` is the firmware for Alpha
  - `roms/seabios` is an implementation of x86 BIOS and VGA BIOS
    `https://git.seabios.org/seabios.git`
  - `roms/seabios-hppa` is a port for parisc/hppa
    `https://github.com/hdeller/seabios-hppa`
  - `roms/sgabios` is an implementation of x86 VGA BIOS over a serial port
  - `roms/skiboot` is OpenPower Abstraction Layer (OPAL) for POWER
    `https://github.com/open-power/skiboot`
  - `roms/u-boot` is a bootloader for ARM, PPC, MIPS, etc.
    `https://gitlab.denx.de/u-boot/u-boot`

## Build

- `mkdir out; cd out`
- `../configure --target-list=x86_64-softmmu --enable-kvm --enable-sdl --enable-opengl --enable-virglrenderer`
  - `enable-sdl` (or gtk)
    - requires sdl (or gtk)
    - enables `-display sdl`
  - `enable-opengl`
    - requires epoxy and gbm
    - enables `-display sdl,gl=on`
  - `enable-virglrenderer`
    - requires virglrenderer
    - enables `-vga virtio`
- `ninja`

## Bootstrap

- `./qemu-img create -f raw disk.img 32G`
- `MACHINE_OPTS="-machine q35 -smp 2 -m 4G"`
- `BLOCK_OPTS="-hda disk.img"`
- `NETWORK_OPTS="-nic user,hostfwd=tcp::2222-:22,model=virtio"`
- `DISPLAY_OPTS="-display sdl,gl=on -vga virtio"`
- `./qemu-system-x86_64 -enable-kvm -cpu host \
   $MACHINE_OPTS $BLOCK_OPTS $NETWORK_OPTS $DISPLAY_OPTS \
   -cdrom <cd>.iso -boot d`
- Arch

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
