Kernel devfreq
==============

## devfreq

- devfreq subsys peridocally polls the busyness of the device and ramps
  up/down the freq based on the busyness
- to use the devfreq subsystem,
  - on init, `devm_devfreq_add_device`
  - on suspend, `devfreq_suspend_device`
  - on resume, `devfreq_resume_device`
- `devm_devfreq_add_device` registers a device to the devfreq subsys
  - it can automatically derive most info from opp
    - `set_freq_table` derives the freq table
    - `scaling_min_freq`, `scaling_max_freq`, `suspend_freq` are from opp
  - it is aware of pm qos, where userspace may change the min/max allowed
    freqs dynamically
  - it requests a default governor
- `devfreq_suspend_device` informs the governor about the event and sets the
  freq to `suspend_freq`
- `devfreq_resume_device` sets the freq to `resume_freq` and informs the
  governor about the event
- `devfreq_dev_profile` describes the device profile
  - `polling_ms` is the interval between polls
  - `get_dev_status` polls total time and busy time (since last poll), and
    current freq
    - the governor uses the info to determine the new freq
  - `target` sets the device to the specified freq
    - an impl typically calls `devfreq_recommended_opp` to return the closest
      opp freq and calls `dev_pm_opp_set_rate` to set the freq
