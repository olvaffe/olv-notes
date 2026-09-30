# Kernel SoC Drivers

## MediaTek

- `CONFIG_MTK_CMDQ`
  - helper to build `cmdq_pkt` for gce mailbox
- `CONFIG_MTK_DEVAPC`
  - mtk bus fabric provides trustzone support
  - unexpected access from master to slave triggers an irq
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
- `CONFIG_MTK_MMSYS` mutex user
  - how `CONFIG_VIDEO_MEDIATEK_MDP3` uses mutex
    - during probe, `mtk_mutex_get` gets `mtk_mutex` from the mutex device for
      each pipeline
      - it relies on dt to find the mutex device
    - `mdp_cmdq_prepare` records cmds for a pipeline
      - `mtk_mutex_prepare` enables clk
      - `mdp_path_config_subfrm`
        - `mdp_path_subfrm_require`
          - `mtk_mutex_write_mod` sets the bits for used hw blocks
          - `mtk_mutex_write_sof` sets SOF to single mode
        - records cmds to write regs
        - `mdp_path_subfrm_run`
          - records cmds to clear SOF
          - `mtk_mutex_enable_by_cmdq` records a cmd to enable mutex
          - records cmds to wait and clear SOF
        - records cmds to wait and clear EOF
    - `mdp_handle_cmdq_callback` is called after cmdq completes
      - `mtk_mutex_unprepare` disables clk
  - it looks like the cmd sequence is
    - `reg write -> clear SOF -> enable mutex -> wait and clear SOF -> wait and clear EOF`
  - i guess
    - reg write is to a shadow reg
    - enable mutex sets SOF
    - disable mutex commits reg write and sets EOF
    - in single mode, enable mutex disables itself automatically
    - in refresh/video mode, mutex is enabled/disabled upon frame start/end

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
- `CONFIG_QCOM_SMEM`
  - fw typically reserves 2MB memory for `qcom,smem`
  - fw stores only partition table in this carveout
    - first 4KB: `struct smem_header`
      - `versions[SMEM_MASTER_SBL_VERSION_INDEX]` is the SBL (secondary
        bootloader) version, which is typically `SMEM_GLOBAL_PART_VERSION`
    - last 4KB: `struct smem_ptable + struct smem_ptable_entry array + struct smem_info`
      - `smem_ptable` is the partition table
        - there is always a global partition
      - `smem_ptable_entry` describes a partition entry
        - each partition is shared by two "hosts"
        - we only care about partitions where one of the hosts is
          `SMEM_HOST_APPS`, ap processor
      - `smem_info`
  - `qcom_smem_get(QCOM_SMEM_HOST_ANY, SBL_MINIDUMP_SMEM_ID)`
    - it calls `qcom_smem_get_private` with the global partition
    - it returns the pointer to the item, which is `minidump_global_toc` in
      this case
- `CONFIG_QCOM_MINIDUMP`
  - <https://lore.kernel.org/lkml/20260708-meminspect-v3-v3-0-7aa5a0a74d5c@oss.qualcomm.com/>
  - `qcom_smem_get` returns the pointer to `minidump_global_toc` on dram
    - `status` is initialized to 1 by sbl
    - `md_version` is 1, which is minidump version
    - `enabled`
  - `minidump_subsystem` array follows `minidump_global_toc`
    - there are several subsystems
      - `MINIDUMP_SUBSYSTEM_APSS` is for ap processor
      - others are for co-processors
    - `qcom_md_table_init` inits `MINIDUMP_SUBSYSTEM_APSS` subsys
      - `status` is set to 1
      - `enabled` is set to `MINIDUMP_SS_ENABLED`
      - `regions_baseptr` is pa of `struct minidump_region` array
        - there can be up to `MAX_NUM_REGIONS` (251) regions
      - `region_count` is initially 0
  - subsystem `struct minidump_region` array
    - `name` is the name of the region, such as `GPU_SNAPSHOT`
    - `valid` is `MINIDUMP_REGION_VALID`
    - `address` is the pa of the region in dram
    - `size` is the size of the region
  - on warm reboot, xbl (extended bootloader) saves the minidump regions to
    persistent storage
    - the persistent storage typically has a `rawdump` partition

## Rockchip

- `CONFIG_ROCKCHIP_GRF`
  - it sets some soc regs to sane defaults
- `CONFIG_ROCKCHIP_IODOMAIN`
  - it sets some io domain regs depending on whether the inputs are 1.8V or 3.3V
