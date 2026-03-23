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

## Qualcomm

- `CONFIG_QCOM_PMIC_GLINK`
  - GLINK is a protocol understood by ADSP FW
  - `pmic_glink_probe` probes the dt-defined pdev
    - `pmic_glink_add_aux_device` creates a dev for `PMIC_GLINK_CLIENT_UCSI`
  - `pmic_glink_rpmsg_probe` probes `PMIC_RTR_ADSP_APPS` rpmsg dev advertised
    by adsp fw
    - it sets up `pg->ept`
    - now the soc glink api can talk to adsp fw over `pg->ept`
      - `devm_pmic_glink_client_alloc`
      - `pmic_glink_client_register`
      - `pmic_glink_send`
  - `CONFIG_UCSI_PMIC_GLINK` is the driver for `PMIC_GLINK_CLIENT_UCSI`
    - `pmic_glink_ucsi_probe` uses the soc glink api to talk to adsp fw
      - this is how we get type-c support on x1e
