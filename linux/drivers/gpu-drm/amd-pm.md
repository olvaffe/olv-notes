DRM AMD PM
==========

## SMU

- `AMD_IP_BLOCK_TYPE_SMC` is based on `MP1_HWIP` version
  - gfx6: `si_smu_ip_block`
  - gfx7: `pp_smu_ip_block` or `kv_smu_ip_block`
  - gfx8: `pp_smu_ip_block`
  - gfx9
    - `pp_smu_ip_block`
    - `smu_v11_0_ip_block`
    - `smu_v12_0_ip_block`
    - `smu_v13_0_ip_block`
  - gfx10
    - `smu_v11_0_ip_block`
    - `smu_v13_0_ip_block`
  - gfx11
    - `smu_v14_0_ip_block`
  - cros
    - skyrim is `IP_VERSION(13, 0, 8)`
- `adev->powerplay.pp_funcs`
  - gfx6: `si_dpm_funcs`
  - gfx7: `pp_dpm_funcs` or `kv_dpm_funcs`
  - gfx8: `pp_dpm_funcs`
  - gfx9: `swsmu_pm_funcs` or `pp_dpm_funcs`
  - gfx10+: `swsmu_pm_funcs`
- in case of swsmu, `smu->ppt_funcs` is also initialized
  - `smu_early_init` calls `smu_set_funcs` to set `ppt_funcs`
    - `IP_VERSION(12, 0, x)` uses `renoir_set_ppt_funcs`
    - `IP_VERSION(11, 5, 0)` uses `vangogh_set_ppt_funcs`
    - `IP_VERSION(13, 0, 8)` uses `yellow_carp_set_ppt_funcs`
  - in fact, `pp_funcs` usually calls `ppt_funcs`
    - e.g., `pp_funcs->set_power_profile_mode` is `smu_set_power_profile_mode`
      which calls `ppt_funcs->set_power_profile_mode`
- swsmu initialization
  - `smu_early_init` creates an `smu_context`
    - `smu->pm_enabled` is based on kernel param `amdgpu_dpm`
    - `pp_funcs` is set to `swsmu_pm_funcs`
    - `smu_set_funcs` inits `ppt_funcs`, `message_map`, `feature_map`,
      `table_map`, etc.
      - `yellow_carp_set_ppt_funcs` for skyrim
      - `smu->od_enabled` stands for overdrive
    - `smu_init_microcode` loads microcode (if any)
  - `smu_sw_init`
    - `smu->power_profile_mode = PP_SMC_POWER_PROFILE_BOOTUP_DEFAULT;`
    - `smu->default_power_profile_mode = PP_SMC_POWER_PROFILE_BOOTUP_DEFAULT;`
    - `smu->workload_mask`, `smu->workload_prority`, and
      `smu->workload_setting` are initialized trivially
    - `smu->smu_dpm.dpm_level = AMD_DPM_FORCED_LEVEL_AUTO;`
    - `smu_smc_table_sw_init`
      - `smu_init_smc_tables` allocs `smu->smu_table`
      - `smu_init_power` allocs `smu->smu_power`
      - `smu_init_fb_allocations` allocs bos for `smu->smu_table`
      - `smu_alloc_memory_pool` allocs bo for `smu->smu_table.memory_pool`
      - `smu_alloc_dummy_read_table` allocs bo for
        `smu->smu_table.dummy_read_1_table`
      - `smu_i2c_init`
  - `smu_hw_init`
    - `smu_start_smc_engine` checks if smc is running
    - `smu_smc_hw_setup` sets up hw
      - `smu_set_driver_table_location` sets `smu->smu_table.driver_table`
      - `smu_set_tool_table_location`
      - `smu_notify_memory_pool_location`
      - `smu_setup_pptable`
      - `smu_write_pptable`
      - `smu_run_btc`
      - `smu_feature_set_allowed_mask`
      - `smu_system_features_control`
      - `smu_init_xgmi_plpd_mode` inits `smu->plpd_mode`
        - to `XGMI_PLPD_NONE` on skyrim
      - `smu_feature_get_enabled_mask` gets enabled features from hw
        - and inits `smu->smu_feature.supported`
      - `smu_is_dpm_running` checks if dpm is running
      - `smu_set_default_dpm_table` gets clock tables from hw and inits
        `smu->smu_table.clocks_table`
      - `smu_update_pcie_parameters`
      - `smu_get_thermal_temperature_range`
      - `smu_enable_thermal_alert`
      - `smu_notify_display_change`
      - `smu_set_min_dcef_deep_sleep`
    - `smu_init_max_sustainable_clocks`
    - `adev->pm.dpm_enabled = true;`
  - `smu_late_init`
    - `smu_set_fine_grain_gfx_freq_parameters` inits
      `smu->gfx_default_hard_min_freq` and `smu->gfx_default_soft_max_freq`
      from `smu->smu_table.clocks_table`
    - `smu_post_init` enables gfxoff
    - `smu_set_ac_dc` informs ac/dc
    - `smu_set_default_od_settings`
    - `smu_populate_umd_state_clk`
    - `smu_get_asic_power_limits`
    - `smu_get_unique_id`
    - `smu_get_fan_parameters`
    - `smu_handle_task(AMD_PP_TASK_COMPLETE_INIT)`
      - `smu_apply_clocks_adjust_rules`
      - `smu_asic_set_performance_level`
        - this is not called because the level is already
          `AMD_DPM_FORCED_LEVEL_AUTO`
      - `smu_bump_power_profile_mode`
        - this is not called because the profile is already
          `PP_SMC_POWER_PROFILE_BOOTUP_DEFAULT`
    - `smu_apply_default_config_table_settings`
    - `smu_restore_dpm_user_profile`
- `enum amd_pm_state_type`
  - commont values are
    - `POWER_STATE_TYPE_DEFAULT`
    - `POWER_STATE_TYPE_POWERSAVE`
    - `POWER_STATE_TYPE_BATTERY`
    - `POWER_STATE_TYPE_BALANCED`
    - `POWER_STATE_TYPE_PERFORMANCE`
  - swsmu does not support this
- `enum PP_SMC_POWER_PROFILE`
  - common values are
    - `PP_SMC_POWER_PROFILE_BOOTUP_DEFAULT`
    - `PP_SMC_POWER_PROFILE_FULLSCREEN3D`
    - `PP_SMC_POWER_PROFILE_POWERSAVING`
    - `PP_SMC_POWER_PROFILE_VIDEO`
    - `PP_SMC_POWER_PROFILE_VR`
    - `PP_SMC_POWER_PROFILE_COMPUTE`
  - swsmu might or might not support this
    - skyrim does not support this
  - `amdgpu_dpm_switch_power_profile` calls `pp_funcs->switch_power_profile`
    which points to `smu_switch_power_profile`
  - `smu_switch_power_profile`
    - when a profile is enabled, the corresponding bit is set in
      `smu->workload_mask`
    - the profile with the highest proirity is chosen
    - `smu_bump_power_profile_mode` sets the profile
- `enum amd_dpm_forced_level`
  - common values are
    - `AMD_DPM_FORCED_LEVEL_AUTO`
    - `AMD_DPM_FORCED_LEVEL_MANUAL`
    - `AMD_DPM_FORCED_LEVEL_LOW`
    - `AMD_DPM_FORCED_LEVEL_HIGH`
    - `AMD_DPM_FORCED_LEVEL_PROFILE_*`
  - ioctl `AMDGPU_CTX_OP_SET_STABLE_PSTATE` can set the profile ones
  - sysfs `power_dpm_force_performance_level` can set any value
  - `amdgpu_dpm_force_performance_level`
    - if a profile one,
      - `amdgpu_device_ip_set_powergating_state`
      - `amdgpu_device_ip_set_clockgating_state`
    - `pp_funcs->force_performance_level` which points to
      `smu_force_performance_level`
  - `smu_force_performance_level`
    - if a profile one, `smu_enable_umd_pstate` toggles
      - `smu_gpo_control`
      - `smu_gfx_ulv_control`
      - `smu_deep_sleep_control`
      - `amdgpu_asic_update_umd_stable_pstate`
        - on skyrim, `nv_update_umd_stable_pstate`
    - `smu_handle_task` calls `smu_adjust_power_state_dynamic`
      - `smu_asic_set_performance_level` sets the freq cap
        - `LOW` caps freqs to min
        - `HIGH` caps freqs to max
        - `AUTO` caps freqs to between min and max
        - `MANUAL` early returns (will be set by other means)
        - `PROFILE_*` caps freqs to either min, max, or a hardcoded default
      - `smu_bump_power_profile_mode` sets the profile to the one of the
        highest priority
- `AMD_DPM_FORCED_LEVEL_MANUAL`
  - sysfs `pp_dpm_*clk`
    - user can pick from the dpm table
    - `smu_print_ppclk_levels` prints the dpm table
    - `smu_force_ppclk_levels` picks from the dpm table
      - this might not be supported
  - `pp_od_clk_voltage`
    - user programs the dpm table directly
    - `amdgpu_dpm_set_fine_grain_clk_vol` is ignored on swsmu
    - `smu_print_ppclk_levels` prints the dpm table
    - `smu_od_edit_dpm_table` edits the dpm table
    - `smu_handle_dpm_task`
- `AMD_DPM_FORCED_LEVEL_PROFILE_*`
  - the profile ones disable clock and power gating
  - `amdgpu_dpm_force_performance_level` calls
    - `amdgpu_device_ip_set_powergating_state`
    - `amdgpu_device_ip_set_clockgating_state`
  - `smu_force_performance_level` calls
    - `amdgpu_asic_update_umd_stable_pstate`
  - on gfx10,
    - `gfx_v10_0_set_powergating_state`
      - `amdgpu_gfx_off_ctrl`
      - `gfx_v10_cntl_pg`
    - `gfx_v10_0_set_clockgating_state`
      - `gfx_v10_0_update_gfx_clock_gating`
    - `nv_update_umd_stable_pstate`
      - `gfx_v10_0_update_perfmon_mgcg`
- gfxoff
  - `amdgpu_gfx_off_ctrl` enables/disables gfxoff
    - it calls `amdgpu_dpm_set_powergating_by_smu` on `AMD_IP_BLOCK_TYPE_GFX`
    - `smu_dpm_set_power_gate` calls `smu_gfx_off_control`
    - `smu_v13_0_gfx_off_control` calls `smu_cmn_send_smc_msg` with
      `SMU_MSG_AllowGfxOff` or `SMU_MSG_DisallowGfxOff`

## DPM

- dpm
  - when a dpm function is called, it usually calls the corresponding function
    in `adev->powerplay.pp_funcs`
    - e.g., `amdgpu_dpm_get_sclk` calls `pp_funcs->get_sclk`
  - newer dpm functions are swsmu-only and call the corresponding `smu->ppt_funcs`
    - e.g., `amdgpu_dpm_mode1_reset` checks `is_support_sw_smu` and calls
      `smu_mode1_reset` which calls `smu->ppt_funcs->mode1_reset`
  - dpm functions fall into 3 categories
    - some are for sysfs
    - some are for amdgpu
    - some are for dc
- dpm funcions used by amdgpu
  - for ioctls
    - `amdgpu_dpm_force_performance_level`, `amdgpu_dpm_get_performance_level`
      - set/get perf level for `AMDGPU_CTX_OP_SET_STABLE_PSTATE` and
        `AMDGPU_CTX_OP_GET_STABLE_PSTATE`
    - `amdgpu_dpm_get_sclk`, `amdgpu_dpm_get_mclk`
      - query sclk/mclk for `AMDGPU_INFO_DEV_INFO`
    - `amdgpu_dpm_get_vce_clock_state`
      - for `AMDGPU_INFO_VCE_CLOCK_TABLE`
    - `amdgpu_dpm_read_sensor`
      - for `AMDGPU_INFO_SENSOR`
  - for ip blocks
    - `amdgpu_dpm_compute_clocks`
      - dc
    - `amdgpu_dpm_enable_jpeg`
      - jpeg
    - `amdgpu_dpm_enable_uvd`
      - vcn/uvd
    - `amdgpu_dpm_enable_vce`
      - vce
    - `amdgpu_dpm_enable_vpe`, `amdgpu_dpm_get_dpm_clock_table`
      - vpe
    - `amdgpu_dpm_set_clockgating_by_smu`
      - clock gating on gfx8
    - `amdgpu_dpm_set_gfx_power_up_by_imu`
      - called from `imu_v11_0_start` on gfx11
    - `amdgpu_dpm_set_powergating_by_smu`
      - enables/disables gfxoff
    - `amdgpu_dpm_smu_i2c_bus_access`
      - used by `smu_v11_0_i2c`
    - `amdgpu_dpm_switch_power_profile`
      - used by amdkfd and vcn
  - reset/suspend/resume
    - `amdgpu_dpm_is_mode1_reset_supported` and `amdgpu_dpm_mode1_reset`
      - `AMD_RESET_METHOD_MODE1`
    - `amdgpu_dpm_mode2_reset`
      - `AMD_RESET_METHOD_MODE2`
    - `amdgpu_dpm_is_baco_supported`, `amdgpu_dpm_baco_enter`,
      `amdgpu_dpm_baco_exit`, `amdgpu_dpm_baco_reset`
      - `AMD_RESET_METHOD_BACO`
    - `amdgpu_dpm_enable_gfx_features`
      - used by `smu_v13_0_10_reset_init`
    - `amdgpu_dpm_gfx_state_change`
      - notify suspend/resume
    - `amdgpu_dpm_notify_rlc_state`
      - called from `amdgpu_device_suspend`
    - `amdgpu_dpm_set_df_cstate`
      - called from `amdgpu_device_ip_suspend`
    - `amdgpu_dpm_set_mp1_state`
      - called from `amdgpu_device_ip_suspend`
    - `amdgpu_dpm_wait_for_event`
      - called from `aldebaran_mode2_restore_ip`
  - debugfs
    - `amdgpu_dpm_get_dpm_freq_range`, `amdgpu_dpm_set_soft_freq_range`
      - for `amdgpu_force_sclk`
    - `amdgpu_dpm_get_residency_gfxoff`, `amdgpu_dpm_set_residency_gfxoff`
      - for `amdgpu_gfxoff_residency`
    - `amdgpu_dpm_get_status_gfxoff`
      - for `amdgpu_gfxoff_status`
      - `dd if=/sys/kernel/debug/dri/0/amdgpu_gfxoff_status status=none bs=4 count=1 | od -A n -t d4`
        - 0 means `GFXOFF(default)`
        - 1 means `Transition out of GFX State`
        - 2 means `Not in GFXOFF`
        - 3 means `Transition into GFXOFF`
    - `amdgpu_dpm_get_entrycount_gfxoff`
      - for `amdgpu_gfxoff_count`
  - others
    - `amdgpu_dpm_handle_passthrough_sbr`
      - for pci passthrough on some asics
    - `amdgpu_dpm_enable_mgpu_fan_boost`
      - multiple dgpu fan boost
    - `amdgpu_dpm_set_xgmi_plpd_mode` and `amdgpu_dpm_set_xgmi_pstate`
      - used by xgmi (multiple dgpu interconnect)
    - `amdgpu_dpm_get_ecc_info`, `amdgpu_dpm_send_hbm_bad_channel_flag`,
      `amdgpu_dpm_send_hbm_bad_pages_num`, `amdgpu_dpm_send_rma_reason`
      - used on memory errors
- clock freqs on swsmu
  - `amdgpu_dpm_get_sclk`, `amdgpu_dpm_get_mclk`,
    `amdgpu_dpm_get_dpm_freq_range`
    - they all call `smu_get_dpm_freq_range` ultimately which looks up in
      `smu->smu_table.clocks_table` for min/max freqs
  - `amdgpu_dpm_set_soft_freq_range`, `amdgpu_dpm_get_sclk_od`,
    `amdgpu_dpm_set_sclk_od`, `amdgpu_dpm_get_mclk_od`,
    `amdgpu_dpm_set_mclk_od`
    - they might no longer be supported
  - `amdgpu_dpm_print_clock_levels`, `amdgpu_dpm_emit_clock_levels`,
    `amdgpu_dpm_force_clock_level`
    - emit is no longer supported; use force instead
    - these print or pick pre-defined levels
    - when picking, it actually calls `smu_set_soft_freq_range` with the
      min/max from the picked levels

## PM sysfs

- `amdgpu_pm_sysfs_init` and `amdgpu_debugfs_pm_init`
- `hwmon_device_register_with_groups` with `hwmon_attributes`
  - temperature
    - `temp*_input`: on-die gpu temperature in millidegrees Celcius
    - `temp*_label`: channel label (edge, junction, and mem)
    - `temp*_crit`
    - `temp*_crit_hyst`
    - `temp*_emergency`
  - voltage
    - `in0_input` and `in0_label`: millivolts on gpu
    - `in1_input` and `in1_label`: millivolts on northbridge
  - power
    - `power*_average`: average power used by gpu in micro-watts
    - `power*_cap_max`: max cap
    - `power*_cap_min`: min cap
    - `power*_cap`: selected cap
    - `power*_cap_default`: default cap
    - `power*_label`: requested power (slowPPT or fastPPT)
  - fan
    - `pwm1`: fan level (0-255)
    - `pwm1_enable`: fan control (0 is no control; 1 manual; 2 auto)
    - `pwm1_min`: min level (0)
    - `pwm1_max`: max level (0)
    - `fan1_input`: cur rpm
    - `fan1_min`: min rpm
    - `fan1_max`: max rpm
    - `fan1_target`: target rpm
    - `fan1_enable`: enable or disable sensor
  - frequency
    - `freq*_input`: gpu (sclk) or mem (mclk) freq in hz
    - `freq*_label`: sclk or mclk
- `amdgpu_device_attr_create_groups` with `amdgpu_device_attrs`
  - `power_dpm_state`: legacy interface
  - `power_dpm_force_performance_level`
    - `auto`
    - `low` forces clocks to lowest freqs
    - `high` forces clocks to highest freqs
    - `manual` allows manual adjs through `pp_dpm_{mclk,sclk,pcie}` and
      `pp_power_profile_mode`
    - `profile_*` disables clock gating and is for profiling
  - `pp_num_states`, `pp_cur_state`, and `pp_force_state`
    - default, battery, balanced, and performance
    - pp stands for PowerPlay
  - `pp_table` sets or shows the current powerplay table
  - `pp_dpm_sclk`, `pp_dpm_mclk`, `pp_dpm_socclk`, `pp_dpm_fclk`,
    `pp_dpm_vclk`, `pp_dpm_dclk`, `pp_dpm_dcefclk`, `pp_dpm_pcie`
    - shows available power levels in the current power state and the freq
    - set `power_dpm_force_performance_level` to `manual` to set
  - `pp_sclk_od`
  - `pp_mclk_od`
  - `pp_power_profile_mode` shows/sets power profiles
    - `3D_FULL_SCREEN`, `VIDEO`, `VR`, `COMPUTE`, `CUSTOM`
  - `pp_od_clk_voltage`
  - `gpu_busy_percent` shows how busy the gpu is
  - `mem_busy_percent` shows how busy VRAM is
  - `pcie_bw` shows pcie bandwidth used
  - `pp_features` shows/sets powerplay features
  - `unique_id` shows the unique id for the gpu
  - `thermal_throttling_logging` logs thermal throttle
  - `gpu_metrics` gives a snapshot of all gpu metrics (temp, freq,
    utilization, etc)
  - `smartshift_apu_power` shows apu power shift in percent
  - `smartshift_dgpu_power` shows dgpu power shift in percent
  - `smartshift_bias` is between -100 (max for apu) to 100 (max for dgpu)
- `amdgpu_debugfs_pm_init` registers `amdgpu_pm_info`
  - it shows all interesting metrics (freqs, voltage, watts, temp, load,
    gating) in human-readable form
- force a frequency for SCLK,
  - echo `manual` to `power_dpm_force_performance_level`
  - cat `pp_dpm_sclk` to see valid power levels and echo the level to select
    - on renoir, `force_clock_level` points to `smu_force_ppclk_levels` which
      calls `renoir_force_clk_levels` to set `SMU_MSG_SetSoftMaxGfxClk`
      `SMU_MSG_SetHardMinGfxClk`
    - there are only 3 levels: hw max, hw min, and 700
      (`RENOIR_UMD_PSTATE_GFXCLK`)
  - or, cat `pp_od_clk_voltage` and echo to reprogram the table
    - for example,
      - `echo 's 0 1500' > pp_od_clk_voltage`
      - `echo 's 1 1500' > pp_od_clk_voltage`
      - `echo 'c' > pp_od_clk_voltage`
    - on renoir, `set_fine_grain_clk_vol` is NULL, `odn_edit_dpm_table` points
      to `smu_od_edit_dpm_table` which calls `renoir_od_edit_dpm_table` to set
      `SMU_MSG_SetSoftMaxGfxClk` and `SMU_MSG_SetHardMinGfxClk`
    - the values are arbitrary, as long as they are between hw min/max
