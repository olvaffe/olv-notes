Kernel cpufreq
==============

## cpufreq

- /sys/devices/system/cpu/cpufreq/policyX
- cur, max, min freq of a CPU
- cur, max, min freq set by the scaling driver
- governor
- benchmarks
  - `dd if=/dev/zero of=/dev/null bs=1M count=10000`
  - `sysbench --time=3 --threads=1 cpu run | grep total`
- for example,
  - `cd /sys/devices/system/cpu`
  - `echo 0 | sudo tee cpu?/online` to bring all but cpu0 offline
  - `cat cpufreq/policy0/cpuinfo_min_freq | sudo tee cpufreq/policy0/scaling_max_freq`
  - this should make a big difference in benchmarks
- <https://www.kernel.org/doc/Documentation/cpu-freq/intel-pstate.txt>
  - `num_pstates` is 32, meaning there are 32 states
  - `turbo_pct` is also 32, meaning there are `32%*num_pstates` in the turbo
    range
  - setting `no_turbo` to 1 makes pstates in the turbo change unavailable
  - `max_perf_pct` is 100 meaning all available pstates can be used
    - i.e., 32 states with turbo or 22 states without turbo
  - `min_perf_pct` is 20 meaning the minimum pstate is 20% of non-turbo
    states
  - by default, it is in active mode with HWP
    - `grep hwp /proc/cpuinfo`
- except `intel_pstate` does not work on an intel machine
  - changing `scaling_max_freq` has no effect
  - changing `max_perf_pct` has no effect
  - and it works now after a reboot....

