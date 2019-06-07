# Latency

## Input

* the kernel is notified of input events by IRQ
  * the kernel driver gets the raw data and do processing/reporting
    * `EV_MSC/MSC_RAW` to report a misc event of raw code
    * `EV_MSC/MSC_SCAN` to report a misc event of keyboard scancode
    * `EV_KEY/keycode` to report a key event of a keycode
    * `EV_SYNC/SYNC_REPORT` to report a sync event
  * common drivers are
    * atkbd device connected to i8042 controller
    * usbhid device connected to usb controller
    * virtio-input device connected to virtio bus
      * the device is described by `virtio_input_config`
      * inputs events are `virtio_input_event` on vq
* the display server receives the kernel input events
  * the display server processes the kernel input events and sends its own
    input events to the client(s)
    * modern display servers use `libinput` to discover input devices and
      handle input events
  * X11 uses XI2 (X Input 2)
    * clients get events such as `XI_ButtonPress` or `XI_KeyPress`
  * Wayland uses `wl_pointer` or `wl_keyboard`
    * clients get events such as `wl_keyboard.key`
* the client receives the display server input events
  * when the client is a hypervisor, it emulates HW input events and generates
    IRQs for the VM.
    * with virtio-input, the event gets translated to a `virtio_input_event`
      and put in the vq.  A VM vcpu thread in the VMM wakes up to handle the
      IRQ.
    * The event travels the same path in the VM again, which doubles the
      latency.
  * when the client is a guest process, the latency overhead can be reduced
    * the guest process functions as a proxy of the host display server
      * it is a client of the host display server
      * it is a server for other guest processes
    * there is only one more hop required (guest display server -> guest
      client)
    * this is possible when the host display server socket is exported to the
      VM, such as using `virtio-vsock`?

## Rendering

* the client responds to the input event, updates its states, and generates
  new frames for the display server
  * the client can also respond to internally generated events, for things
    such as animation
* each new frame should be generated onto a buffer shared with the display
  server
  * the client should "acquire" one of the shared buffers.  This is needed
    because the client needs to make sure the server is not using the same
    buffer.
  * the client renders to the buffer.  It can be done with CPU.  Or it can
    be done with GPU, video decoder, or a camera.
  * the client should "release" the buffer back.  This tells the server that
    the buffer is ready for presentation
* When the buffer is rendered with GPU or similar devices, the rendering does
  not complete after the client submits the rendering job to the device and is
  done with the buffer.  Quite the contrary, the rendering just begins.
  * in the "release" event, the client wants to send the server a "fence"
    object which is signaled when the rendering completes.  In the case with
    CPU rendering, the fence is already signaled when it is sent.
  * we want this kind of pipelining because by the time the server picks up
    the buffer, the rendering might already complete.  Even better, when the
    servier presents the buffer using device "coherent" with the rendering
    device, the fence becomes irrelevant.
  * because the server can also sample the buffer using a GPU or a similar
    device, when the server is done with the buffer, the sampling might just
    begin.  When the client "acquires", the client should also receive a
    "fence" object which is signaled when the sampling completes.

## Output

* the display server receives a client frame
  * server in the host
  * server in the guest
* the kernel receives a page flip
  * vsync
