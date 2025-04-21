DRM msm display
===============

## DT

- USB nodes
  - `x1e80100.dtsi`
    - `usb_1_ss0: usb@a6f8800` is USB 2.0/3.0 controller
      - `usb_1_ss0_hsphy: phy@fd3000` is 2.0 phy
      - `usb_1_ss0_qmpphy: phy@fd5000` is 3.0 phy
        - this also supports altmode from `mdss_dp0: displayport-controller@ae90000`
    - `usb_1_ss1: usb@a8f8800` is USB 2.0/3.0 controller
      - `usb_1_ss1_hsphy: phy@fd9000` is 2.0 phy
      - `usb_1_ss1_qmpphy: phy@fda000` is 3.0 phy
        - this also supports altmode from `mdss_dp1: displayport-controller@ae98000`
    - `usb_1_ss2: usb@a0f8800` is USB 2.0/3.0 controller
      - `usb_1_ss2_hsphy: phy@fde000` is 2.0 phy
      - `usb_1_ss2_qmpphy: phy@fdf000` is 3.0 phy
        - this also supports altmode from `mdss_dp2: displayport-controller@ae9a000`
    - `usb_2: usb@a2f8800` is USB 2.0 controller
      - `usb_2_hsphy: phy@88e0000` is 2.0 phy
    - `usb_mp: usb@a4f880` is USB 2.0/3.0 controller
      - `usb_mp_hsphy0: phy@88e1000` is 2.0 phy
      - `usb_mp_hsphy1: phy@88e2000` is 2.0 phy
      - `usb_mp_qmpphy0: phy@88e3000` is 3.0 phy
      - `usb_mp_qmpphy1: phy@88e5000` is 3.0 phy
  - `x1-asus-vivobook-s15.dtsi`
    - `usb_1_ss0` supports 2.0/3.0/altmode
      - the 3.0 phy connects to a retimer (because of altmode?)
    - `usb_1_ss1` supports 2.0/3.0/altmode
      - the 3.0 phy connects to a retimer (because of altmode?)
    - `usb_1_ss2` is unused
    - `usb_2` supports 2.0
      - the 2.0 phy connects to a repeater
    - `usb_mb` supports 2.0/3.0
      - the 2.0 phys connect to repeaters
- Display nodes
  - `x1e80100.dtsi`
    - `mdss: display-subsystem@ae00000` is MDSS subsystem
      - `mdss_mdp: display-controller@ae01000` is DPU
        - it outputs to DP n
      - `mdss_dp0: displayport-controller@ae90000` is DP 0
        - it takes an input from DPU
        - it outputs to `usb_1_ss0_qmpphy: phy@fd5000`
      - `mdss_dp1: displayport-controller@ae98000` is DP 1
        - it takes an input from DPU
        - it outputs to `usb_1_ss1_qmpphy: phy@fda000`
      - `mdss_dp2: displayport-controller@ae9a000` is DP 2
        - it takes an input from DPU
        - it outputs to `usb_1_ss2_qmpphy: phy@fdf000`
      - `mdss_dp3: displayport-controller@aea0000` is DP 3
        - it takes an input from DPU
        - it outputs to built-in display
  - `x1-asus-vivobook-s15.dtsi`
    - `mdss_dp0` is enabled for altmode
    - `mdss_dp1` is enabled for altmode
    - `mdss_dp2` is disabled
    - `mdss_dp3` is enabled for built-in display
      - `aux-bus/panel` is the panel

## MDSS

- componentized device
  - when a subdevice is probed, add it with `component_add`
  - when the master device is probed, describe the required subdevices and
    add the master with `component_master_add_with_match`
    - it will attempt to bind the master device to its driver
    - the driver init function calls `component_bind_all` to bind the
      subdevices to their respective drivers
- `msm_drm_init` is the driver init function for the master device
  - it initializes MDSS with `dpu_mdss_init` first
  - it then binds all subdevices
  - the binding of DPU initializes KMS with `dpu_bind`
  - it then calls kms's `hw_init`
  - it starts a `crtc_commit:%d` and a `crtc_event:%d` thread for each crtc
  - `drm_vblank_init`

## DisplayPort

- `msm_dp_display_probe` provides the DP controllers
  - `msm_dp_display_get_connector_type` determines DP or eDP
    - if the node has a `aux-bus/panel` subnode, it is eDP
  - if eDP, `devm_of_dp_aux_populate_bus` adds the panel to `dp-aux` bus
    - the panel driver will bind to the panel
  - `msm_dp_display_probe_tail` calls `devm_drm_of_get_bridge` to get the
    bridge from endpoint 0 port 1
    - if eDP, this returns the panel
    - if DP, this returns the USB 3.0 phy nowadays
