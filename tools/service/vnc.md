VNC
===

## History

- RFB, Remote Frame Buffer, is the protocol used by VNC, Virtual Network
  Computing
  - RFB by default uses tcp port 5900
  - the server and client negotiate encodings
- RealVNC
  - VNC was originally developed by ORL, Olivetti & Oracle Research Lab
  - RealVNC was formed after the closure of ORL in 2002
  - later versions no longer FOSS?
- TightVNC
  - supports "tight" encoding which is very efficient
  - later versions no longer FOSS?
- TurboVNC
  - forked from TightVNC
  - focus on 3D
- TigerVNC
  - forked from TightVNC
  - provides
    - `Xvnc` is both an xserver and a vnc server
    - `x0vncserver` is a vnc server for an existing xserver
    - `vncconfig` configures a running vnc server
    - `vncpasswd` updates vnc password in `~/.vnc/passwd`
    - `vncsession` starts `Xvnc`, for use with systemd
    - `vncviewer` is a vnc client
- LibVNC provides an RFB implementation for both the server and the client
- noVNC is an html-based vnc client

## kmsvnc

- <https://chromium.googlesource.com/chromiumos/platform2/+/refs/heads/main/screen-capture-utils/>
  - uses `LibVNCServer` to provide a vnc server
- `CrtcFinder::Find` scans drm crtcs and returns a `Crtc`
- `EglDisplayBuffer` wraps a `Crtc`
- `Uinput` uses `uinput` to provide input emulation
- mainloop
  - `EglDisplayBuffer::Capture`
    - `WaitVBlank` calls `drmWaitVBlank` to wait for the next vblank
    - `GetConnectedPlanes` calls `drmModeGetFB2` on all planes to get the
      current fbs
    - `CreateImage` exports the plane fb as dma-buf and imports it as an EGL
      image
    - the fbs are drawn into an fbo
    - `glReadPixels` reads back the pixels
  - `ConvertBuffer` packs the pixels tightly
  - `server->frameBuffer` is updated to point to pixels
  - `rfbProcessEvents` sends pixels to the client
