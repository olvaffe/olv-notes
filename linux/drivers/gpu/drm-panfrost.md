DRM panfrost
===========

## PM

- panfrost supports multiple power domains, clocks, and regulators
  - `panfrost_pm_domain_init`
    - it bails if there is no or a single power domain, because
      `platform_probe` already takes care
    - `dev_pm_domain_attach_by_name`
    - `device_link_add`
  - `panfrost_devfreq_init`
    - it bails (disables devfreq) if there is more than one regulator
    - otherwise, the regulator (and the clk) is managed by opp
  - `panfrost_regulator_init`
    - it is called only when devfreq is disabled
    - `devm_regulator_bulk_get`
    - `regulator_bulk_enable`
  - iow,
    - panfrost supports 0, 1, or more power domains
    - panfrost supports 1, or 2 clks (more clks are ignored)
    - panfrost supports 1, or more regulators
      - it enables devfreq only when there is 1 regulator
      - newer kernel always uses 1 regulator, by using a regulator coupler
- the device is powered on by default
  - `panfrost_probe` calls `pm_runtime_set_active` before `pm_runtime_enable`
- `panfrost_pm_ops` is the `dev_pm_ops`
  - `panfrost_device_runtime_suspend` is called when pm deems the device idle
    - this suspends devfreq, irq, and powers off some blocks
    - this is called from `genpd_runtime_suspend`, which powers off genpd
  - `panfrost_device_runtime_resume`
    - this is called from `genpd_runtime_resume`, which powers on genpd
    - this resets the gpu, powers on some blocks, and resumes devfreq and irq
  - `panfrost_device_suspend` is called before system suspend
    - `pm_runtime_force_suspend` forces `panfrost_device_runtime_suspend`
    - `GPU_PM_CLK_DIS` disables clks
    - `GPU_PM_VREG_OFF` disables regulators (managed by opp)
    - later, `genpd_suspend_noirq` powers off genpd
  - `panfrost_device_resume` is called after system resume
    - earlier, `genpd_resume_noirq` powers on genpd
    - if `GPU_PM_VREG_OFF`, enables regulators (managed by opp)
    - if `GPU_PM_CLK_DIS`, enables clks
    - `pm_runtime_force_resume` forces `panfrost_device_runtime_resume`
