Linux V4L2
==========

## Core

- drivers call `v4l2_device_register` to register a `v4l2_device` in their
  probe functions
  - this is not exposed to userspace
- drivers also call `video_register_device` to register one or more
  `video_device`
  - depending on `vfl_devnode_type`, this creates
    - `/dev/videoX` for `VFL_TYPE_VIDEO`
    - `/dev/vbiX` for `VFL_TYPE_VBI`
    - `/dev/radioX` for `VFL_TYPE_RADIO`
    - `/dev/v4l-subdevX` for `VFL_TYPE_SUBDEV`
    - `/dev/swradioX` for `VFL_TYPE_SDR`
    - `/dev/v4l-touch` for `VFL_TYPE_TOUCH`
  - `v4l2_fops` is the `file_operations`
- `v4l2_fops`
  - `v4l2_open` calls `v4l2_file_operations::open`
    - `video_devdata` uses the inode to look up `video_device` from `file`
    - drivers use `video_drvdata` to look up driver private data
      - for uvc, the private data is `uvc_streaming`
      - uvc also allocates a `uvc_fh` and saves it to `file->private_data`
  - `v4l2_ioctl` calls `v4l2_file_operations::unlocked_ioctl`
    - it is usually the generic `video_ioctl2` which dispatches using
      `v4l2_ioctls`

## Query ioctls

- `VIDIOC_QUERYCAP`
  - `V4L2_CAP_VIDEO_CAPTURE` captures signals and stores them in memory
    - e.g., camera, tv card, etc.
  - `V4L2_CAP_VIDEO_OUTPUT` reads pixels from memory and outputs signals
    - e.g., tv broadcast, signal transcoding (when supporting both capture and
      output)
  - `V4L2_CAP_VIDEO_OVERLAY` injects an overlay into display output signal?
  - `V4L2_CAP_VBI_CAPTURE` captures signals inside vblank intervals and stores
    them in memory
    - e.g., those signals are usually teletext or closed caption
  - `V4L2_CAP_VBI_OUTPUT` reads pixels from memory and outputs signals in
    vblank intervals
  - `V4L2_CAP_SLICED_VBI_CAPTURE` captures and demodulates VBI signals
  - `V4L2_CAP_SLICED_VBI_OUTPUT` modulates and outputs VBI signals
  - `V4L2_CAP_RDS_CAPTURE` captures RDS (Radio Data System) info in radio
    signals
  - `V4L2_CAP_VIDEO_OUTPUT_OVERLAY` outputs signals by reading and overlaying
    multiple sources
    - the 2nd source is exposed as a fb device
  - `V4L2_CAP_HW_FREQ_SEEK` seeks radio freq in hw
  - `V4L2_CAP_RDS_OUTPUT` outputs RDS info in radio signals
  - `V4L2_CAP_VIDEO_CAPTURE_MPLANE` captures in multi-plane formats
    - this means multi-plane formats while `V4L2_CAP_VIDEO_CAPTURE` means
      single-plane formats
  - `V4L2_CAP_VIDEO_OUTPUT_MPLANE` outputs from multi-plane formats
    - this means multi-plane formats while `V4L2_CAP_VIDEO_OUTPUT` means
      single-plane formats
  - `V4L2_CAP_VIDEO_M2M_MPLANE` supports memory-to-memory operations such as
    codec, post-processing, format coversion, etc.
    - it supports both output (sending data from mem to hw) and capture
      (receiving data from hw to mem) in the conventional sense
  - `V4L2_CAP_VIDEO_M2M` is the same but with single-plane formats
  - `V4L2_CAP_TUNER` demodulates video/radio signals
  - `V4L2_CAP_AUDIO` is additional audio inputs/output for video
    capture/output
  - `V4L2_CAP_RADIO` receives/transmits radio signals
  - `V4L2_CAP_MODULATOR` modulates video/radio signals
  - `V4L2_CAP_SDR_CAPTURE` receives SDR (software-defined radio) signals
  - `V4L2_CAP_EXT_PIX_FORMAT` supports extended fields in `v4l2_pix_format`
    - when `priv` is set to `V4L2_PIX_FMT_PRIV_MAGIC`, all fields following
      `priv` are valid
  - `V4L2_CAP_SDR_OUTPUT` transmits SDR signals
  - `V4L2_CAP_META_CAPTURE` captures non-image data associated with video
    signals
  - `V4L2_CAP_READWRITE` supports capturing with `read` syscall and outputing
    with `write` syscall
  - `V4L2_CAP_STREAMING` supports zero-copy streaming io
    - rather than copying data between userspace and kernel space, this
      supports using `V4L2_MEMORY_MMAP` or `V4L2_MEMORY_DMABUF` to achieve
      zero-copy
  - `V4L2_CAP_META_OUTPUT` outputs non-image data along with video signals
  - `V4L2_CAP_TOUCH` works similar to `V4L2_CAP_VIDEO_CAPTURE`, but the hw
    captures signals that are from a touch sensor
    - this is for touch devices to report raw data rather than input events to
      userspace
  - `V4L2_CAP_IO_MC` means the device should not be used directly but
    controlled via a media controller
  - `V4L2_CAP_DEVICE_CAPS` means `device_caps` is initialized
    - `device_caps` is a subset of `capabilities`, which gives the caps of the
      current dev node (e.g., `/dev/video0`)
    - `capabilities` gives the caps of the hw, which can expose multiple dev
      nodes
- `VIDIOC_ENUM_FMT`, `VIDIOC_ENUM_FRAMESIZES`, and
  `VIDIOC_ENUM_FRAMEINTERVALS`
  - the `video_device` has an array of supported formats for each buftype
    - for each supported format, there is an array of supported frame sizes
      (resolutions)
    - for each supported format and resolution, there is an array of supported
      frame intervals (framerates)
  - these ioctls enumerate the format arrays
- `VIDIOC_ENUMINPUT`
  - the `video_device` may support multiple input sources
  - this ioctl enumerates input sources

## Stateful ioctls

- `VIDIOC_G_FMT` and `VIDIOC_G_FMT`
  - the `video_device` has the concept of "current format" for each buftype
    - this includes pix format, width, height, stride, etc.
  - these ioctls query or set the current formats
- `VIDIOC_G_PARM` and `VIDIOC_S_PARM`
  - this is mainly used to set capture interval (framerate)
  - the `video_device` has the concept of "current interval" for each buftype
- `VIDIOC_G_INPUT` and `VIDIOC_S_INPUT`
  - the hw might support multiple inputs
  - these query or set the current input

## Buffer ioctls

- `VIDIOC_CREATE_BUFS`
  - it dispatches to driver `vidioc_create_bufs` which usually calls
    `vb2_create_bufs`
  - `enum v4l2_memory`
    - `V4L2_MEMORY_MMAP` means kernel allocates buffers that userspace can
      mmap
    - `V4L2_MEMORY_DMABUF` means kernel imports dmabufs from usersapce?
  - `__vb2_buf_mem_alloc` makes the allocation
    - `call_ptr_memop(alloc, ...)` calls `vb2_vmalloc_alloc` when the
      `mem_ops` is `vb2_vmalloc_memops`
    - the buffer is allocated by `vmalloc_user`
- `VIDIOC_EXPBUF`
  - it dispatches to driver `vidioc_expbuf` which usually calls
    `vb2_expbuf`
  - the memory type must be `V4L2_MEMORY_MMAP`
  - the `mem_ops` must support `get_dmabuf`
    - all `vb2_vmalloc_memops`, `vb2_dma_sg_memops`, and
      `vb2_dma_contig_memops` support `get_dmabuf`
    - e.g., uvc uses `vb2_vmalloc_memops`

## Old

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
