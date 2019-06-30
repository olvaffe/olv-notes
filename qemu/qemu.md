# Get Source Code

 - git clone https://git.qemu.org/git/qemu.git
 - cd qemu
 - git submodule init
 - git submodule update --recursive

# Build

 - ./configure \
    --target-list=x86_64-softmmu \
    --enable-kvm \
    --enable-sdl \
    --enable-opengl \
    --enable-virglrenderer
   - enable-sdl (or gtk)
     - requires sdl (or gtk)
     - enables "-display sdl"
   - enable-opengl
     - requires epoxy and gbm
     - enables "-display sdl,gl=on"
   - enable-virglrenderer
     - requires virglrenderer
     - enables "-vga virtio"
 - make

# Bootstrap

 - ./qemu-img create -f qcow2 <disk>.qcow2 35G
 - MACHINE_OPTS="-machine q35 -smp 2 -m 4G"
 - BLOCK_OPTS="-hda <disk>.qcow2"
 - NETWORK_OPTS="-nic user,hostfwd=tcp::2222-:22,model=virtio"
 - DISPLAY_OPTS="-display sdl,gl=on -vga virtio"
 - ./x86_64-softmmu/qemu-system-x86_64 -enable-kvm -cpu host \
    $MACHINE_OPTS $BLOCK_OPTS $NETWORK_OPTS $DISPLAY_OPTS \
    -cdrom <cd>.iso -boot d
 - Arch

# Tips

 - release mouse grab
   - Ctrl-Alt or
   - Ctrl-Alt-G
 - SSH
   - ssh ssh://root@localhost:2222

# Starup

* most things in qemu are modules
* modules are registered before `main()` using `register_module_init`
  * each module type is added to a different list
* when qemu enters main, it calls `module_call_init` to initialize modules
* for a QOM (qemu object model) module, the initialization routine registers
  the object type with `type_register_static`
  * this adds the type to the typesystem
  * the type can be looked up using the type name
* for a OPTS module, the initialization routine registers `QemuOptsList` with
  `qemu_add_opts`
  * the core also registers many `QemuOptsList` itself
  * they are used to parse top-level options
* all top-level options are defined in `qemu-options.def`
  * to parse an option, qemu finds the corresponding registered `QemuOptsList`
    and calls `qemu_opts_parse`
  * it parses the option into a `QemuOpts` that can be looked up and used
    later
* qemu then picks a machine class
  * it gets a list of all registered machine types in QOM
  * it looks at the parsed machine option to pick the desired machine class
  * it initializes the QOM class
* qemu goes on to do many things, mainly to configure the machine according to
  the specified options
  * one of them is to add devices specified with device options to the machine
* when the machine configuration is finalized, qemu calls
  `machine_run_board_init` to initialize
  * this calls `pc_q35_init` for q35 machine
  * it adds a bunch more essential devices (each of a QOM type) to the machine

# Display

* Before main, type `virtio-vga` is registered to the QOM
* `-vga virtio` selects the `virtio-vga` as the VGA type for the machine
* `-display` does not use QOM or `QemuOpts`
  * displays are registered with `qemu_display_register`
  * the option is parsed to `DisplayOptions`
* once a display type is selected, the display and the console are early
  initalized with `qemu_display_early_init` and `qemu_console_early_init`
* when the q35 machine is initialized in `machine_run_board_init`, it calls
  `pc_vga_init`
  * this creates the `virtio-vga` device
  * it calls `virtio_vga_base_realize`, and indirect calls
    `virtio_gpu_device_realize` to realize it vgpu object
  * virtio-gpu registers a graphic console with `graphic_console_init`
* finally `qemu_display_init` is called
  * this creates the window for the VM

# virtio-input

* `-device virtio-keyboard-pci` and ` -device virtio-mouse-pci`
  * when devices are added to the machine, the options add the desired PCI devices
  * each of them own a `virtio-{keyboard,mouse}-device`
  * they register input handlers with `qemu_input_handler_register`
  * when the display gets inputs, it invokes the handlers.  This gives them a
    chance to inject the events to the device and sends IRQs
* when a key is pressed, it is processed by the display (SDL2/gtk/...) and
  `qkbd_state_key_event` is called
  * it goes a long way but eventually `qemu_input_event_send_impl` is called
