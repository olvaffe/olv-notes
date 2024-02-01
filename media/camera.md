Camera
======

## Simple Command Tools

- `cam` uses `libcamera`
  - `cam --list` to list cameras
  - `cam --camera 1 --list-controls --list-properties --info` to show controls,
    properties, and info of camera 1
    - controls are `Brightness`, `Contrast`, `AeEnable`, `ExposureTime`, etc.
    - properties are `Model`, `PixelArraySize`, etc.
    - info is `1280x720-MJPEG`, `1280x720-YUYV`, etc.
  - `cam --camera 1 --capture --F` to capture frames and save them to files
    - specify `--stream key=val,...` such as `--stream pixelformat=MJPEG` to
      configure the stream
  - `cam --camera 1 --capture --sdl` to capture frames and display them using SDL
- `yavta` talks to V4L2 directly
  - `yavta --list-controls --enum-formats --enum-inputs /dev/video0` to show
    info, controls, formats, and inputs of `/dev/video0`
  - `yavta --capture --file` to capture frames and save them to files
- `v4l2-ctl` talks to V4L2 directly
  - `v4l2-ctl --list-devices` to list devices
  - `v4l2-ctl --device /dev/video0 --info --list-ctrls` to show info and
    controls of `/dev/video0`
  - `v4l2-ctl --device /dev/video0 --list-inputs` to list inputs
  - `v4l2-ctl --device /dev/video0 --list-formats-ext` to list formats
  - `v4l2-ctl --device /dev/video0 --stream-count=1 --stream-mmap --stream-to=frame.raw --verbose`
    to capture a frame and save it to `frame.raw`
    - specify `--verbose --set-fmt-video pixelformat=MJPG` to use MJPEG, if
      supported
    - if the format is `YUYV`,
      - `ffmpeg -f rawvideo -s 1280x720 -pix_fmt yuyv422 -i frame.raw result.png`
