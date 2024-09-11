Kernel Mailbox
==============

## Controllers

- a mailbox is a mechanism for inter-processor communication
  - between the ap processor and various coprocessors
- controller driver
  - allocates a `mbox_controller`
  - allocates a `mbox_chan` for each channel
  - inits controller and channel structs
    - especially `mbox_chan_ops`
  - `devm_mbox_controller_register` to register

## Consumers

- consumer driver
  - `mbox_request_channel` to request a `mbox_chan`
  - `mbox_send_message` to send a request
    - the controller will forward the request to the coprocessor attached to
      the channel
    - when a response is available, `rx_callback` is called

## MT8195 Examples

- `mediatek/mt8195.dtsi`
  - `gce0: mailbox@10320000`
    - `compatible = "mediatek,mt8195-gce";`
  - `vdosys0: syscon@1c01a000`
    - `mboxes = <&gce0 0 CMDQ_THR_PRIO_4>;`
  - `vdosys1: syscon@1c100000`
    - `mboxes = <&gce0 1 CMDQ_THR_PRIO_4>;`
- `CONFIG_MTK_CMDQ_MBOX` enables the CMDQ mailbox driver for
  `mediatek,mt8195-gce`
- drm mediatek uses a mailbox
  - `&gce0` references the CMDQ driver
  - `mbox_request_channel` returns a channel to CMDQ mailbox
  - `mbox_send_message` calls `cmdq_mbox_send_data`
- in another example,
  - `CONFIG_MTK_ADSP_MBOX` enables the ADSP (audio dsp) mailbox driver for
    `mediatek,mt8195-adsp-mbox`
  - `CONFIG_SND_SOC_SOF_MT8195` enables the ADSP SoF driver for
    `mediatek,mt8195-dsp`
    - it registers the `mtk-adsp-ipc` platform device
  - `CONFIG_MTK_ADSP_IPC` enables the ADSP protocol driver for `mtk-adsp-ipc`
    - `mbox_request_channel_byname` returns a channel to ADSP mailbox
    - when the sof driver calls `mtk_adsp_ipc_send`, `mbox_send_message` calls
      `mtk_adsp_mbox_send_data`
