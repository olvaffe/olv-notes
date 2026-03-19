Kernel SoC Drivers
==================

## MediaTek

- `CONFIG_MTK_CMDQ`
  - helper to build `cmdq_pkt` for gce mailbox
- `CONFIG_MTK_DEVAPC`
  - mtk bus fabric provides trustzone support
  - unexpected access from master to slave triggers an irq
      - select `MediaTek CMDQ Support`
      - select `MediaTek DVFSRC Support`
- `CONFIG_MTK_DVFSRC`
  - dvfsrc can collect utilization from hw blocks and control their voltage
    and frequency automatically
  - it can also take sw input via `mtk_dvfsrc_send_request`
