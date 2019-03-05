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
 - ./x86_64-softmmu/qemu-system-x86_64 -enable-kvm \
    $MACHINE_OPTS $BLOCK_OPTS $NETWORK_OPTS $DISPLAY_OPTS \
    -cdrom <cd>.iso -boot d
 - Arch

# Tips

 - release mouse grab
   - Ctrl-Alt or
   - Ctrl-Alt-G
 - SSH
   - ssh ssh://root@localhost:2222
