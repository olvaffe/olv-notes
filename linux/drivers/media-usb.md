Linux media usb
===============

## UVC Initialization

- `uvc_probe` probes the UVC device
  - `uvc_parse_control` parses UVC control descriptor
    - `uvc_parse_standard_control` adds `streams` and `entities`
  - `v4l2_device_register` registers a `v4l2_device`
  - `uvc_scan_device` groups `entities` into `chains`
  - `uvc_ctrl_init_device` adds ctrls to entities
  - `uvc_register_chains` calls `uvc_register_video` to register a
    `video_device` for each stream
- `uvc_parse_streaming` parses UVC control descriptor and adds a
  `uvc_streaming`
  - `streaming->type` is either `V4L2_BUF_TYPE_VIDEO_OUTPUT` or
    `V4L2_BUF_TYPE_VIDEO_CAPTURE`
  - `streaming->formats` is initialized
    - note that `uvc_format` has an `uvc_frame` array, and `uvc_frame` has an
      `dwFrameInterval` array
    - these are for `uvc_ioctl_enum_framesizes` and
      `uvc_ioctl_enum_frameintervals` queries
