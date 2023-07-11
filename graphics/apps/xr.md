XR (VR, AR, MR)
===============

## APIs

- OpenXR
  - <https://registry.khronos.org/OpenXR/>
- OpenVR
  - <https://github.com/ValveSoftware/openvr>
- ARKit
  - <https://developer.apple.com/augmented-reality/arkit/>
- ARCore
  - <https://developers.google.com/ar/develop>
- Oculus
- PlayStation VR

## Headsets

- Meta Quest Pro
  - 2022
  - standalone / tethered
  - 1800x1920 per-eye
  - supported apis: oculus, openxr
- Meta Quest 2
  - 2020
  - standalone / tethered
  - 1832x1920 per-eye
  - supported apis: oculus, openxr
- VIVE XR Elite
  - 2023
  - standalone / tethered
  - 1920x1920 per-eye
  - supported apis: openxr
- HP Reverb G2
  - 2022
  - tethered
  - 2160x2160 per-eye
  - supported apis: openxr, openvr
- Valve Index
  - 2019
  - tethered
  - 1440x1600 per-eye
  - supported apis: openxr, openvr
- Tethered, aka, PC VR
  - app runs on pc
  - headset is used as an input/output device
  - it can be wired
    - quest link
  - or wireless
    - quest air link
    - alvr

## Engines

- Godot
  - <https://docs.godotengine.org/en/stable/tutorials/xr/index.html>
  - supports: openxr, openvr, arkit
- Unity
  - <https://docs.unity3d.com/Manual/XR.html>
  - supports: openxr, oculus, arkit, arcore, playstation
  - 3rd-party supports: openvr
- Unreal
  - <https://www.unrealengine.com/en-US/xr>

## Linux

- traditional tethered HMDs show up as displays on the host machine
  - the kernel driver marks them as non-desktop
    - `EDID_QUIRK_NON_DESKTOP` for some
    - `drm_parse_microsoft_vsdb` for those who follow windows
    - `update_displayid_info` for those with displayid
  - the compositor ignores them
    - wlroots marks them as `non_display` and sway ignores them
  - the app (usually the vr compositor) leases them
    - app uses `wp_drm_lease_device_v1` to lease them
    - app uses `VK_EXT_acquire_drm_display` to get `VkDisplayKHR`
    - app uses `VK_KHR_display` and `VK_EXT_direct_mode_display` to manage
      them
- newer tethered HMDs show up as USB devices on the host machine
  - they speak proprietary protocols and require drivers to work
  - drivers are usually openxr drivers
  - apps are written against openxr api
- Monado
  - <https://gitlab.freedesktop.org/monado/monado>
  - it is similar to gallium but for headsets
  - it has state trackers, aka frontends
    - openxr
    - openvr
  - it has drivers
    - `daydream` for google daydream (2016)
    - `hdk` for OSVR HDK (2016)
    - `hydra` for Razer Hydra controller (2011)
    - `illixr` for Illinois Extended Reality testbed for researchers  (2021)
    - `north_star` for Leap Motion Project North Star (2018)
    - `ohmd` is an adapter for OpenHMD, which supports a bunch of (old)
      devices
    - `opengloves` for OpenGlove vr gloves
    - `psmv` for PlayStation Move controller (2018)
    - `pssense` for PlayStation VR2 Sense controller (2023)
    - `psvr` for PlayStation VR headset (2016)
    - `qwerty` for simulation through mouse and keyboard
    - `realsense` for intel RealSense tracking camera (2015)
    - `remote` listens to a tcp port
    - `rift_s` for Oculus Rift S (2019)
    - `simula` for Simula VR computer
    - `steamvr_lh` is an adapter for SteamVR's driver for Vive Lighthouse Base
      Stations
    - `survive` is an adatper for libsurvive, an open-source driver for Vive
      Lighthouse Base Stations
    - `ultraleap_v2` for Leap Motion v2
    - `v4l2` is an adapter for V4L2
    - `vf` decodes a video file
    - `vive` for various vive devices
    - `wmr` for windows mixed reality headsets
