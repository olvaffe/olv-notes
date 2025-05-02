Latency
=======

## Input

- the kernel is notified of input events by IRQ
  - the kernel driver gets the raw data and do processing/reporting
    - `EV_MSC/MSC_RAW` to report a misc event of raw code
    - `EV_MSC/MSC_SCAN` to report a misc event of keyboard scancode
    - `EV_KEY/keycode` to report a key event of a keycode
    - `EV_SYNC/SYNC_REPORT` to report a sync event
  - common drivers are
    - atkbd device connected to i8042 controller
    - usbhid device connected to usb controller
    - virtio-input device connected to virtio bus
      - the device is described by `virtio_input_config`
      - inputs events are `virtio_input_event` on vq
- the display server receives the kernel input events
  - the display server processes the kernel input events and sends its own
    input events to the client(s)
    - modern display servers use `libinput` to discover input devices and
      handle input events
  - X11 uses XI2 (X Input 2)
    - clients get events such as `XI_ButtonPress` or `XI_KeyPress`
  - Wayland uses `wl_pointer` or `wl_keyboard`
    - clients get events such as `wl_keyboard.key`
- the client receives the display server input events
  - when the client is a hypervisor, it emulates HW input events and generates
    IRQs for the VM.
    - with virtio-input, the event gets translated to a `virtio_input_event`
      and put in the vq.  A VM vcpu thread in the VMM wakes up to handle the
      IRQ.
    - The event travels the same path in the VM again, which doubles the
      latency.
  - when the client is a guest process, the latency overhead can be reduced
    - the guest process functions as a proxy of the host display server
      - it is a client of the host display server
      - it is a server for other guest processes
    - there is only one more hop required (guest display server -> guest
      client)
    - this is possible when the host display server socket is exported to the
      VM, such as using `virtio-vsock`?

## Rendering and Explicit Fencing

- the client responds to the input event, updates its states, and generates
  new frames for the display server
  - the client can also respond to internally generated events, for things
    such as animation
- each new frame should be generated onto a buffer shared with the display
  server
  - the client should "acquire" one of the shared buffers.  This is needed
    because the client needs to make sure the server is not using the same
    buffer.
  - the client renders to the buffer.  It can be done with CPU.  Or it can
    be done with GPU, video decoder, or a camera.
  - the client should "release" the buffer back.  This tells the server that
    the buffer is ready for presentation
- When the buffer is rendered with GPU or similar devices, the rendering does
  not complete after the client submits the rendering job to the device and is
  done with the buffer.  Quite the contrary, the rendering just begins.
  - in the "release" event, the client wants to send the server a "fence"
    object which is signaled when the rendering completes.  In the case with
    CPU rendering, the fence is already signaled when it is sent.
  - we want this kind of pipelining because by the time the server picks up
    the buffer, the rendering might already complete.  There will be no
    waiting at all.  Also, when the server presents the buffer using a device
    "coherent" with the rendering device, there will be no waiting either.
  - because the server can also sample the buffer using a GPU or a similar
    device, when the server is done with the buffer, the sampling might just
    begin.  When the client "acquires", the client should also receive a
    "fence" object which is signaled when the sampling completes.

## Output

- the display server receives a client frame
  - the client frame may still be busy.  Inspect its fence smartly.
  - the server uses a single scanout buffer
    - It can copy the client frame content to the scanout buffer immediately
      (w/ tearing) or on next vsync (no tearing).  Its returned fence signals
      when the copying is completed.
  - the server page flips between two scanout buffers
    - flip between front and back scanout buffers on vsyncs
    - It can copy the client frame content to the back scanout buffer
      immediately (no tearing).  Its returned fence signals when the copying
      is completed.
  - the client is full screen or the server supports overlays
    - the client frame can be made the scanout buffer (or the source buffer of
      an overlay) on next vsync.  The server's returned fence signals when
      there is another buffer replacing the client frame as the scanout buffer
      (or the source buffer).
  - the client is in a guest and connects to the server in the host using a
    proxy in the guest.
    - the client frame reachs the proxy first before reaching the real server
      in the host
    - fences in both directions also need to be relayed
- the kernel receives a page flip or atomic commit request
  - kernel returns immediately and return a fence
  - the HW flips to the new buffer on next vsync
    - this can be simulated in SW using a thread
  - kernel signals the fence
    - the fence signals when the display controller starts reading from the
      new buffer, not finishes reading.  In other words, the previous frame
      becomes idle.
  - with `virtio-gpu`, the flip of buffers translates to
    `VIRTIO_GPU_CMD_SET_SCANOUT`.  The hypervisor makes the new buffer the
    texture for scanout.  That is, it blits from the new buffer to the host
    window, and does a swap buffer on the host window.
  - with `virtio-gpu`, the copying to scanout buffer translates to
    `VIRTIO_GPU_CMD_RESOURCE_FLUSH`.  The hypervisor makes sure the texture is
    scanned out again.  That is, it blits from the current buffer to the host
    window, and does a swap buffer on the host window.

## Measuring Latency

- glxgears X traffic
     000:>:0065: Event KeyPress(2) keycode=0x41 time=0x001bb075 root=0x0000039c event=0x00400002 child=None(0x00000000) root-x=984 root-y=384 event-x=174 event-y=-6 state=Mod2 same-screen=true(0x01)
     000:<:0066: 72: Present-Request(146,1): Pixmap window=0x00400002 pixmap=0x0040000a serial=3 valid=0x00000000 update=0x00000000 x_off=0 y_off=0 target_crtc=0x00000000 wait_fence=0x00000000 idle_fence=0x0040000b options=0 target_msc=25769803776 divisor=0 remainder=0 notifies=;
     000:>:0066: Event Generic(35) Present(146) IdleNotify(2) event=0x00400005 window=0x00400002 serial=2 pixmap=0x00400008 idle_fence=0x00400009
     000:>:0066: Event Generic(35) Present(146) CompleteNotify(1) kind=Pixmap(0x00) mode=Copy(0x00) event=0x00400005 window=0x00400002 serial=3 ust=7790846842723368960 msc=38654705664
     (Flip instead of Copy when fullscreen)
- X main loop calls `Dispatch`
  - `WaitForSomething`
    - at the beginning, it calls `BlockHandler`
    - if there is nothing to do, it sleeps
    - after it wakes up, it calls `WakeupHandler`
- Example (glxgears on host)
  - +0us:   input IRQ (i8042)
  - +136us: X wake up to send the input event
  - +356us: glxgears wake up
  - +421us: glxgears handles the input event
  - +436us: glxgears calls GL
  - +738us: glxgears swap buffers (which flushes) to present the frame on next msc
  - +905us: X wake up and waits for the next vsync
  - some time later
  - +0us: vsync IRQ (i915)
  - +63us: X wake up again to copy from app buffer to scanout buffer
  - new content shows up tear-free
- Example (glxgears in guest, traced in guest, on a different host than above)
  - +   0us: input IRQ (i8042)
  - + 180us: X wake up to send the input event
  - + 341us: glxgears wake up
  - + 380us: glxgears handles the input event
  - + 395us: glxgears calls GL
  - + 516us: glxgears swap buffers (which flushes) to present the frame on next msc
  - + 723us: X wake up (no wait for vsync because virtio-gpu does not support
      it)
  - + 821us: X submits copy
  - +1663us: X submits copy returns
    - it take a while because `virtqueue_kick` (`vp_notify` specifically)
      writes to the notification register.  If host has pending works for us,
      they are fired before the write returns
  - +1719us: X flushes resource (dirty fb)
- Example (glxgears in guest, traced in host)
  - +   0us: input IRQ (i8042)
  - + 151us: X wake up to send the input event
  - after a while
  - +   0us: qemu wake up
    - qemu wakes up every `GUI_REFRESH_INTERVAL_DEFAULT` (30) ms, or
      `SDL2_REFRESH_INTERVAL_BUSY` (10) ms.
  - + 287us: qemu `handle_keydown`
  - + 313us: qemu generates an input IRQ for the guest
  - +1561us: guest notifies host about glxgears GPU command with MMIO write
  - +1782us: qemu process guest glxgears GPU command
  - +2253us: guest notifies host about X copy GPU command with MMIO write
  - +3250us: qemu `i915_request_add` for the guest glxgears
  - +3794us: qemu generates a cmd done IRQ (qemu polls GL sync every 10ms)
  - +4116us: qemu process guest X GPU command
  - +4482us: guest notifies host about `RESOURCE_FLUSH` GPU command with
             MMIO write
  - +4723us: qemu `i915_request_add` for the guest X
             (no IRQ is generated in the captured timeframe)
  - +5565us: qemu process guest `RESOURCE_FLUSH` GPU command
  - +5988us: qemu `i915_request_add` to blit from guest scanout to window
  - +6230us: X wake up
  - +7089us: X `i915_request_add` to copy
  - Note that qemu produces a new frame every 30ms-ish and creates noises
  - Note that guest fences are translated to GPU commands on guest kernel 5.2
    or older.  That creates noises.
- Example (glxgears in crosvm, traced in crosvm)
  - when there is any client, X wake up every second
    - call block handlers twice
      - call xwayland block handler
      - call glamor block handler
      - glFlush
  - glxgears
    - +0us: virtio-wl irq
    - +488us: sommilier wake up
      - it `VIRTWL_IOCTL_RECV`, demarshal the data to wayland traffic to a
      	socket, read the wayland traffic, and send it to X
    - +721us: sommilier sends the wayland traffic to X
    - +920us: X wake up and enter `socket_handler` to read wl events and send
              X events
    - +1617us: X enters block handlers and glamor `glFlush`
      - twice!
    - +1787us: glxgears wake up
    - +2238us: glxgears queues a GPU command
    - +4090us: X wake up again and flips
    - +4281us: sommilier wake up
      - `sl_host_surface_attach`, it seems
        - +6574us: `DRM_IOCTL_PRIME_FD_TO_HANDLE`
        - +6627us: `DRM_VIRTGPU_WAIT`
        - +6634us: `DRM_IOCTL_GEM_CLOSE`
    - +6975us: sommilier `VIRTWL_IOCTL_SEND`
- Example (glxgears in crosvm, traced in host)
  - idle (guest X wakes up every second)
    - +0us  : vcpu thread wake up
    - +248us: vgpu wake up
    - +355us: i915 execbuffer
    - +769us: vcpu again
    - +893us: vgpu again
    - +978us: i915 again
  - glxgears
    - +0us    : irq (i8042)
    - +180us  : chrome evdev thread wake up
    - +263us  : chrome main thread wake up (exo?)
    - +576us  : crosvm wl thread wake up
    - +686us  : virtio-wl irq generated in host
    - +1072us : virtio-wl irq acked by guest
    - guest doing its work
    - +7453us : virtio-gpu wake up
    - +7503us : virtio-gpu execbuf (dummy for X)
    - +8068us : virtio-gpu execbuf (dummy for X)
    - +8394us : virtio-gpu execbuf (for glxgears)
    - +8677us : virtio-wl mmio (`guest->host` notification)
    - +8684us : crosvm wl thread wake up
    - +8702us : chrome main thread wake up
      - +8794us: chrome io thread wake up
      - it wakes up ChildIOT in chrome gpu process
    - +8908us : chrome gpu process wake up
    - +8956us : chrome main thread wake up
    - +10962us: chrome gpu process wake up
    - +11009us: chrome gpu process execbuffer
    - +11194us: virtio-gpu execbuf (?)
    - +11364us: chrome DrmThread wake up
    - +11396us: `DRM_IOCTL_MODE_ATOMIC`
    - +11445us: `i915_pipe_update_start`
    - +11455us: `i915_pipe_update_end`
    - some time later: vblank and flip complete
- Interesting events
  - input: irq -> server -> client
  - client: input -> state update -> acquire -> render -> present
  - output: client -> server -> pageflip -> vsync
- ftrace
  - `sched/sched_switch`: which process is running
  - `irq`: input irq
  - `drm`: vblank and pageflip-complete events
  - `dma_fence`: when is fence signaled
  - `trace_marker`: client events
- set `trace_clock` (no need to resort to kvm ptp?)
