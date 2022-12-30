V4L2 buffers

application calls VIDIOC_REQBUFS to request/query the number of buffers and
access method.  Suppose the access method is V4L2_MEMORY_MMAP.  App calls
VIDIOC_QUERYBUF on each buffer to get the offset and length of the buffer.  It
then mmap /dev/video0 for each offset/length pairs given.  Take
videbuf-dma-sg.c for example, the mmap caused the baddr (userspace address of
the buffer) to be set.

In kernel, each buffer has a statte.  The initial state is VIDEOBUF_NEEDS_INIT.
Before the buffers could be used, app should call VIDIOC_QBUF.  The specified
buffer is then initialized (if needed), prepared (state VIDEOBUF_PREPARED), and
queued (state VIDEOBUF_QUEUED).  If there is no current buffer, it is activated
(state VIDEOBUF_ACTIVE).  Take videbuf-dma-sg.c for example again, the
initialization calls get_user_pages on the mapped area to pagefault (allocate)
pages at once.  The frames are later scatter-gathered to them.

A call to VIDIOC_STREAMON starts the streaming.  After the current buffer is
filled, it changes state to VIDEOBUF_DONE and /dev/video0 becomes readable.  To
get the contents, the app calls VIDIOC_DQBUF (state VIDEOBUF_IDLE) and copy
directly through the userspace address of the buffer.  At this stage, the
buffer is queued again through VIDIOC_QBUF.

See SOC_CAMERA in Kconfig or pxa_camera.c/sh_mobile_ceu_camera.c for more
info on soc host and mt9* on soc device.


SOC CAMERA
A device is registered through soc_camera_device_register:

soc_camera_device_register -> (bus probe) soc_camera_probe -> (host add) -\                   ->(device probe) -> (host remove) -\
                                                                           -> (device init) -/                                    -> (device release)

and is unregistered through soc_camera_device_unregister:

soc_camera_device_unregister -> (bus remove) soc_camera_remove -> (device remove)

On /dev/video0 is opened:

soc_camera_open -> (host add for first instance) \> (device init) /> (host init_videobuf)

VIDIOC_QUERYCAP -> soc_camera_querycap -> (host querycap)
VIDIOC_ENUMSTD -> video_ioctl2
VIDIOC_ENUMINPUT -> soc_camera_enum_input
VIDIOC_ENUM_FMT -> soc_camera_enum_fmt_vid_cap
VIDIOC_G_FMT -> soc_camera_g_fmt_vid_cap
VIDIOC_QUERYCTRL -> soc_camera_queryctrl
VIDIOC_G_CONTROL -> soc_camera_g_ctrl -> (device get_control)
VIDIOC_G_CHIP_IDENT -> soc_camera_g_chip_ident -> (device get_chip_id)
VIDIOC_STREAMON -> soc_camera_streamon -> (device start_capture)


? -> soc_camera_s_fmt_vid_cap -> (host try_bus_param) -> (host try_fmt_cap) \> (device query_bus_param) /> (host set_fmt_cap) \> (device set_fmt_cap) /> (host set_bus_param) \> (device query_bus_param) /> (device set_bus_param)
