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
- `CONFIG_MTK_SOCINFO`
  - `CONFIG_NVMEM_MTK_EFUSE` creates a child device named `mtk-socinfo`
  - this driver binds to the child device and expects the parent to have two
    nvmem cells, `socinfo-data1` and `socinfo-data2`
  - it then uses the cell vals to identify the soc and exposes human-redable
    info via `/sys/devices/soc%d`
- `CONFIG_MTK_SVS`
  - the hw calculates optimal voltages for cpu and gpu based on calibration
    data, current temperature, etc.
  - the driver then calls `dev_pm_opp_adjust_voltage` to adjust opp voltages
  - it seems to "undervolt" cpu and gpu to save power

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

## Rockchip

- `CONFIG_ROCKCHIP_GRF`
  - it sets some soc regs to sane defaults
- `CONFIG_ROCKCHIP_IODOMAIN`
  - it sets some io domain regs depending on whether the inputs are 1.8V or 3.3V
